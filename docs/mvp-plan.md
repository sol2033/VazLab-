# План разработки MVP: Диагностика Lada

## 🎯 Цель MVP
Создать мобильное приложение (Android в приоритете) для диагностики автомобилей Lada, которое:
- Подключается к ELM327 адаптеру через Bluetooth
- Читает коды ошибок (DTC) с ЭБУ
- Отображает описания ошибок на русском языке
- Позволяет сбрасывать ошибки
- Имеет простой и понятный интерфейс

## 🛠️ Технологический стек
- **Frontend**: Flutter (Dart)
- **Платформа**: Android (первая версия), затем iOS, Windows
- **Связь**: Bluetooth Serial (ELM327)
- **База данных**: SQLite (локальное хранение)
- **Архитектура**: Provider / Riverpod для state management

## 📋 Этапы разработки MVP

---

### Фаза 0: Настройка окружения (1-3 дня)

**Цель**: Подготовить среду разработки и структуру проекта

#### Задачи:
- Установить Flutter SDK (последняя stable версия)
- Настроить Android Studio с Flutter plugin
- Настроить эмулятор Android или подключить физическое устройство
- Проверить `flutter doctor` - все должно быть ✅
- Создать новый Flutter проект
- Настроить Git репозиторий
- Добавить базовые зависимости в pubspec.yaml

**Зависимости для MVP:**
```yaml
dependencies:
  flutter:
    sdk: flutter
  
  # Bluetooth
  flutter_bluetooth_serial: ^0.4.0
  permission_handler: ^11.0.0
  
  # State management
  provider: ^6.1.0
  
  # База данных
  sqflite: ^2.3.0
  path_provider: ^2.1.0
  
  # UI
  google_fonts: ^6.1.0
  
  # Утилиты
  intl: ^0.18.0
```

**Структура проекта:**
```
lada_diagnostics/
├── android/
├── ios/
├── lib/
│   ├── main.dart
│   ├── core/
│   │   ├── constants/
│   │   │   ├── obd_commands.dart
│   │   │   └── dtc_codes.dart
│   │   └── utils/
│   │       ├── hex_parser.dart
│   │       └── validators.dart
│   ├── models/
│   │   ├── dtc_model.dart
│   │   ├── ecu_info.dart
│   │   └── connection_state.dart
│   ├── services/
│   │   ├── elm327_service.dart
│   │   ├── bluetooth_service.dart
│   │   └── dtc_database_service.dart
│   ├── providers/
│   │   ├── connection_provider.dart
│   │   └── diagnostics_provider.dart
│   ├── screens/
│   │   ├── home_screen.dart
│   │   ├── connection_screen.dart
│   │   ├── diagnostics_screen.dart
│   │   └── settings_screen.dart
│   └── widgets/
│       ├── device_list_item.dart
│       ├── dtc_card.dart
│       ├── connection_status.dart
│       └── custom_button.dart
├── assets/
│   ├── data/
│   │   └── dtc_database.json
│   └── images/
├── test/
└── pubspec.yaml
```

**Критерий готовности:**
- Проект создан и запускается на устройстве/эмуляторе
- Все зависимости установлены
- Git инициализирован

---

### Фаза 1: Bluetooth подключение (3-5 дней)

**Цель**: Реализовать сканирование и подключение к ELM327 через Bluetooth

#### Задачи:

**1.1 Bluetooth Service (bluetooth_service.dart)**
- Запрос разрешений на Bluetooth и местоположение (Android 12+)
- Сканирование доступных Bluetooth устройств
- Фильтрация устройств (поиск ELM327, OBD по имени)
- Подключение к выбранному устройству
- Отключение и управление соединением
- Обработка ошибок подключения

**Ключевые функции:**
```dart
class BluetoothService {
  Future<List<BluetoothDevice>> scanDevices()
  Future<bool> connectToDevice(String address)
  Future<void> disconnect()
  Stream<String> getDataStream()
  Future<void> sendCommand(String command)
  bool get isConnected
}
```

**1.2 Connection Screen (connection_screen.dart)**
- UI для сканирования устройств
- Список найденных Bluetooth устройств
- Индикатор загрузки при сканировании
- Кнопка "Обновить список"
- Статус подключения
- Обработка разрешений (запрос при необходимости)

**UI элементы:**
- AppBar с заголовком "Подключение"
- FloatingActionButton для сканирования
- ListView с найденными устройствами
- Индикатор подключения/отключения

**1.3 Connection Provider (connection_provider.dart)**
- State management для состояния подключения
- Управление списком устройств
- Хранение информации о текущем подключенном устройстве
- Уведомления об изменении состояния

**Критерий готовности:**
- Приложение сканирует Bluetooth устройства
- Можно подключиться к ELM327
- Отображается статус подключения
- Обрабатываются ошибки подключения

---

### Фаза 2: ELM327 протокол (4-6 дней)

**Цель**: Реализовать базовую работу с ELM327 командами

#### Задачи:

**2.1 OBD Commands константы (obd_commands.dart)**
```dart
class OBDCommands {
  // AT команды
  static const String RESET = 'ATZ';
  static const String ECHO_OFF = 'ATE0';
  static const String SPACES_OFF = 'ATS0';
  static const String HEADERS_OFF = 'ATH0';
  static const String AUTO_PROTOCOL = 'ATSP0';
  static const String GET_PROTOCOL = 'ATDPN';
  static const String VOLTAGE = 'ATRV';
  
  // OBD команды
  static const String GET_DTC = '03';
  static const String CLEAR_DTC = '04';
  static const String PENDING_DTC = '07';
  static const String GET_VIN = '0902';
  
  // Mode 01 (живые данные)
  static const String ENGINE_RPM = '010C';
  static const String VEHICLE_SPEED = '010D';
  static const String COOLANT_TEMP = '0105';
  static const String THROTTLE_POS = '0111';
}
```

**2.2 ELM327 Service (elm327_service.dart)**

Основные методы:
```dart
class ELM327Service {
  // Инициализация адаптера
  Future<bool> initialize()
  
  // Базовые команды
  Future<String> sendCommand(String command)
  Future<String> getVersion()
  Future<double> getVoltage()
  Future<String> detectProtocol()
  
  // Диагностика
  Future<List<String>> readDTC()
  Future<bool> clearDTC()
  Future<List<String>> readPendingDTC()
  
  // Парсинг ответов
  List<String> parseDTCResponse(String response)
  String parseHexResponse(String response)
}
```

**Логика инициализации ELM327:**
1. Отправить `ATZ` (reset)
2. Ждать ответа (2-3 секунды)
3. Отправить `ATE0` (отключить эхо)
4. Отправить `ATL0` (отключить переносы строк)
5. Отправить `ATS0` (отключить пробелы)
6. Отправить `ATH0` (отключить заголовки)
7. Отправить `ATSP0` (автоопределение протокола)
8. Проверить связь с ЭБУ командой `0100`

**Парсинг DTC кодов:**
```dart
// Пример ответа: 43 01 33 00 00 00 00
// 43 - ответ на команду 03
// 01 - количество кодов
// 33 00 - код ошибки P0133 (первые 2 бита + 4 hex цифры)

String parseDTC(String hexCode) {
  int firstByte = int.parse(hexCode.substring(0, 2), radix: 16);
  int secondByte = int.parse(hexCode.substring(2, 4), radix: 16);
  
  String prefix = ['P', 'C', 'B', 'U'][firstByte >> 6];
  String code = prefix + (firstByte & 0x3F).toRadixString(16).toUpperCase() 
                + secondByte.toRadixString(16).padLeft(2, '0').toUpperCase();
  
  return code;
}
```

**2.3 Hex Parser (hex_parser.dart)**
- Очистка ответов от лишних символов
- Преобразование hex в decimal
- Валидация ответов

**Критерий готовности:**
- Приложение инициализирует ELM327
- Можно прочитать коды ошибок
- Коды правильно парсятся (P0133, P0171 и т.д.)
- Можно сбросить ошибки

---

### Фаза 3: База кодов ошибок (2-3 дня)

**Цель**: Создать базу описаний DTC кодов для ВАЗ

#### Задачи:

**3.1 DTC Database JSON (dtc_database.json)**

Структура:
```json
{
  "standard_codes": {
    "P0100": {
      "description": "Неисправность цепи датчика массового расхода воздуха (ДМРВ)",
      "symptoms": "Неустойчивая работа двигателя, повышенный расход",
      "possible_causes": [
        "Неисправен датчик ДМРВ",
        "Подсос воздуха",
        "Проблемы с проводкой"
      ],
      "severity": "medium"
    },
    "P0133": {
      "description": "Медленный отклик датчика кислорода 1, банк 1",
      "symptoms": "Повышенный расход топлива, нестабильные холостые",
      "possible_causes": [
        "Загрязнен лямбда-зонд",
        "Требуется замена датчика кислорода"
      ],
      "severity": "medium"
    },
    "P0171": {
      "description": "Слишком бедная топливовоздушная смесь, банк 1",
      "symptoms": "Потеря мощности, неустойчивая работа",
      "possible_causes": [
        "Подсос воздуха",
        "Неисправен ДМРВ",
        "Низкое давление топлива",
        "Загрязнены форсунки"
      ],
      "severity": "high"
    },
    "P0172": {
      "description": "Слишком богатая топливовоздушная смесь, банк 1",
      "symptoms": "Черный дым из выхлопной, повышенный расход",
      "possible_causes": [
        "Неисправен ДМРВ",
        "Негерметичны форсунки",
        "Неисправен датчик кислорода"
      ],
      "severity": "high"
    },
    "P0300": {
      "description": "Обнаружены случайные/множественные пропуски зажигания",
      "symptoms": "Вибрация двигателя, потеря мощности",
      "possible_causes": [
        "Неисправны свечи зажигания",
        "Проблемы с катушкой зажигания",
        "Низкая компрессия"
      ],
      "severity": "high"
    }
  },
  "vaz_specific_codes": {
    "P1602": {
      "description": "Пропадание напряжения бортовой сети ЭБУ (ВАЗ)",
      "symptoms": "Загорается Check Engine",
      "possible_causes": [
        "Плохой контакт",
        "Проблемы с проводкой ЭБУ",
        "Низкое напряжение АКБ"
      ],
      "severity": "low",
      "ecu": ["Январь 7.2", "Bosch M7.9.7"]
    },
    "P0363": {
      "description": "Пропуски воспламенения с отключением подачи топлива (ВАЗ)",
      "symptoms": "Двигатель троит",
      "possible_causes": [
        "Неисправны свечи",
        "Катушка зажигания",
        "Форсунки"
      ],
      "severity": "high",
      "ecu": ["Январь 7.2", "Bosch M7.9.7"]
    }
  }
}
```

**3.2 DTC Model (dtc_model.dart)**
```dart
class DTCModel {
  final String code;
  final String description;
  final String symptoms;
  final List<String> possibleCauses;
  final String severity; // low, medium, high
  final DateTime? detectedAt;
  final bool isPending;
  
  DTCModel({...});
  
  factory DTCModel.fromJson(Map<String, dynamic> json);
  Map<String, dynamic> toJson();
  
  // Получить цвет по severity
  Color getSeverityColor() {
    switch(severity) {
      case 'high': return Colors.red;
      case 'medium': return Colors.orange;
      default: return Colors.yellow;
    }
  }
}
```

**3.3 DTC Database Service (dtc_database_service.dart)**
```dart
class DTCDatabaseService {
  Map<String, dynamic> _database = {};
  
  Future<void> loadDatabase()
  DTCModel? getCodeInfo(String code)
  List<String> searchCodes(String query)
  bool isVAZSpecific(String code)
}
```

**Критерий готовности:**
- База содержит минимум 50 стандартных кодов
- База содержит минимум 20 ВАЗ-специфичных кодов
- Все коды имеют русские описания
- Сервис правильно загружает и парсит базу

---

### Фаза 4: UI диагностики (3-4 дня)

**Цель**: Создать интерфейс для отображения и работы с ошибками

#### Задачи:

**4.1 Diagnostics Screen (diagnostics_screen.dart)**

**Компоненты экрана:**
- AppBar с заголовком и статусом подключения
- Кнопка "Сканировать ошибки"
- Список найденных ошибок (если есть)
- Индикатор загрузки при сканировании
- Кнопка "Сбросить все ошибки"
- Placeholder когда ошибок нет ("Ошибки не обнаружены ✅")

**4.2 DTC Card Widget (dtc_card.dart)**

**Элементы карточки:**
```dart
class DTCCard extends StatelessWidget {
  final DTCModel dtc;
  
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Заголовок с кодом и иконкой severity
            Row(
              children: [
                Icon(Icons.error, color: dtc.getSeverityColor()),
                Text(dtc.code, style: boldStyle),
                Spacer(),
                Chip(label: Text(dtc.severity)),
              ],
            ),
            
            // Описание
            Text(dtc.description, style: titleStyle),
            
            // Симптомы
            if (dtc.symptoms.isNotEmpty)
              ExpansionTile(
                title: Text("Симптомы"),
                children: [Text(dtc.symptoms)],
              ),
            
            // Возможные причины
            ExpansionTile(
              title: Text("Возможные причины"),
              children: dtc.possibleCauses.map((cause) => 
                ListTile(
                  leading: Icon(Icons.chevron_right),
                  title: Text(cause),
                )
              ).toList(),
            ),
          ],
        ),
      ),
    );
  }
}
```

**4.3 Home Screen (home_screen.dart)**

**Главный экран приложения:**
- Карточка статуса подключения
  - Зеленая если подключено
  - Красная если отключено
  - Показывает имя устройства
- Кнопка "Подключиться" или "Отключиться"
- Кнопка "Диагностика" (активна только при подключении)
- Кнопка "Настройки"
- Информация о приложении (версия)

**4.4 Diagnostics Provider (diagnostics_provider.dart)**
```dart
class DiagnosticsProvider extends ChangeNotifier {
  List<DTCModel> _dtcList = [];
  bool _isScanning = false;
  String? _errorMessage;
  
  List<DTCModel> get dtcList => _dtcList;
  bool get isScanning => _isScanning;
  bool get hasErrors => _dtcList.isNotEmpty;
  
  Future<void> scanForErrors()
  Future<void> clearAllErrors()
  Future<void> clearSingleError(String code)
}
```

**Критерий готовности:**
- UI отображает список ошибок
- Карточки ошибок содержат всю информацию
- Можно сбросить все ошибки
- Есть индикация загрузки
- Обрабатываются ошибки (нет связи, таймаут)

---

### Фаза 5: Тестирование и полировка (2-3 дня)

**Цель**: Протестировать на реальном автомобиле и исправить баги

#### Задачи:

**5.1 Реальное тестирование**
- Подключение к вашему Lada
- Чтение реальных ошибок
- Проверка правильности парсинга
- Тестирование сброса ошибок
- Проверка стабильности подключения
- Тест на разных расстояниях от адаптера (Bluetooth)

**5.2 Обработка edge cases**
- Потеря соединения во время сканирования
- Адаптер не отвечает
- ЭБУ не поддерживает команду
- Нет ошибок в памяти
- Слишком много ошибок (>10)
- Неизвестные коды ошибок

**5.3 UI/UX улучшения**
- Добавить темную тему
- Улучшить анимации переходов
- Добавить тултипы и подсказки
- Onboarding при первом запуске (3 экрана)
  1. "Подключите ELM327 к автомобилю"
  2. "Включите зажигание"
  3. "Разрешите доступ к Bluetooth"
- Иконки для разных типов severity
- Звуковая обратная связь (опционально)

**5.4 Настройки (settings_screen.dart)**
- Выбор языка (RU/EN)
- Автоподключение к последнему устройству
- Таймаут команд (по умолчанию 5 сек)
- Очистка кэша
- О приложении (версия, лицензия)

**5.5 Обработка ошибок и логирование**
```dart
class ErrorHandler {
  static void handle(dynamic error, {String? context}) {
    // Логирование ошибки
    print('Error in $context: $error');
    
    // Показ пользователю
    if (error is TimeoutException) {
      return 'Таймаут подключения. Проверьте адаптер.';
    } else if (error is BluetoothException) {
      return 'Ошибка Bluetooth. Проверьте настройки.';
    } else {
      return 'Неизвестная ошибка. Попробуйте снова.';
    }
  }
}
```

**Критерий готовности:**
- Приложение работает на реальном автомобиле
- Все основные сценарии протестированы
- Критические баги исправлены
- UI выглядит профессионально

---

### Фаза 6: Сборка и подготовка к релизу (1-2 дня)

**Цель**: Собрать APK и подготовить к распространению

#### Задачи:

**6.1 Подготовка манифеста и разрешений**

**AndroidManifest.xml:**
```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<application
    android:label="Lada Диагностика"
    android:icon="@mipmap/ic_launcher">
    ...
</application>
```

**6.2 Иконка приложения**
- Создать иконку (1024x1024)
- Адаптивная иконка для Android
- Использовать flutter_launcher_icons package

**6.3 Сборка релизной версии**
```bash
# Создать keystore
keytool -genkey -v -keystore ~/key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias key

# Настроить key.properties
storePassword=<пароль>
keyPassword=<пароль>
keyAlias=key
storeFile=/path/to/key.jks

# Собрать APK
flutter build apk --release

# Или AAB для Google Play
flutter build appbundle --release
```

**6.4 Документация**
- README.md с инструкциями
- Список поддерживаемых ЭБУ
- FAQ по использованию
- Список совместимых адаптеров ELM327
- Скриншоты приложения

**Критерий готовности:**
- APK успешно собран
- Приложение устанавливается на устройство
- Все функции работают в релизной версии
- Размер APK оптимизирован (<15 МБ)

---

## 📊 Общая временная оценка MVP

| Фаза | Длительность | Приоритет |
|------|-------------|-----------|
| 0. Настройка окружения | 1-3 дня | Критично |
| 1. Bluetooth подключение | 3-5 дней | Критично |
| 2. ELM327 протокол | 4-6 дней | Критично |
| 3. База кодов ошибок | 2-3 дня | Высокий |
| 4. UI диагностики | 3-4 дня | Высокий |
| 5. Тестирование | 2-3 дня | Критично |
| 6. Сборка релиза | 1-2 дня | Средний |

**Итого:** 16-26 дней (3-5 недель при работе 4-6 часов в день)

---

## 🎯 Функционал MVP (Минимальный)

### ✅ Что БУДЕТ в MVP:
1. ✅ Подключение по Bluetooth к ELM327
2. ✅ Чтение активных кодов ошибок (Mode 03)
3. ✅ Отображение описаний ошибок на русском
4. ✅ Сброс всех ошибок (Mode 04)
5. ✅ Базовая информация об адаптере (версия, напряжение)
6. ✅ Простой и понятный UI
7. ✅ Темная тема
8. ✅ Поддержка Android

### ❌ Что НЕ будет в MVP (следующие версии):
- ❌ Чтение живых параметров (RPM, температура и т.д.)
- ❌ Логирование данных
- ❌ AI-объяснения ошибок
- ❌ Графики и аналитика
- ❌ Экспорт отчетов
- ❌ USB и Wi-Fi подключение
- ❌ iOS версия
- ❌ Windows/Desktop версия
- ❌ Проприетарные ВАЗ-PID

---

## 🚀 После MVP: Roadmap v1.1-v1.3

### v1.1 - Живые параметры (2-3 недели)
- Чтение основных параметров (Mode 01)
- Dashboard с карточками параметров
- Цветовая индикация норм/отклонений

### v1.2 - Логирование (2 недели)
- Запись параметров во время поездки
- Просмотр логов с графиками
- Экспорт в CSV

### v1.3 - Расширенная поддержка (3 недели)
- USB подключение (для десктопа)
- Wi-Fi ELM327
- iOS версия
- Проприетарные PID для ВАЗ

---

## 📚 Полезные ресурсы

### Документация:
1. **ELM327 Datasheet** - основной reference по AT командам
2. **OBD-II PIDs** - Wikipedia, полный список стандартных PID
3. **ISO 14230-4 (KWP2000)** - протокол для Bosch ЭБУ
4. **ISO 15765-4 (CAN)** - протокол для современных ЭБУ

### Форумы и сообщества:
1. **chiptuner.ru/forum** - основной русский форум по ВАЗ ЭБУ
2. **forum.autoscan.ru** - форум по автодиагностике
3. **drive2.ru** - сообщество владельцев ВАЗ
4. **4pda.to** - форум Android разработки

### Инструменты:
1. **Torque Pro** - референс приложение для тестирования
2. **OBD Auto Doctor** - desktop приложение для диагностики
3. **ELM327 Identifier** - проверка версии адаптера
4. **Bluetooth Terminal** - отладка команд ELM327

---

## 🔍 Критерии успеха MVP

### Технические:
- ✅ Успешное подключение к ELM327 в 95% случаев
- ✅ Корректное чтение DTC кодов
- ✅ Время инициализации <10 секунд
- ✅ Стабильность: работа без сбоев 30+ минут
- ✅ Размер APK <15 МБ

### Пользовательские:
- ✅ Интуитивно понятный интерфейс (не требует инструкции)
- ✅ Быстрая работа (отклик UI <100ms)
- ✅ Понятные сообщения об ошибках
- ✅ Описания кодов на русском языке

### Бизнес:
- ✅ Приложение решает основную задачу: чтение и сброс ошибок
- ✅ Готово к тестированию реальными пользователями
- ✅ Есть база для дальнейшего развития

---

## 📝 Следующие шаги

1. **Начать с Фазы 0** - настроить окружение
2. **Протестировать ELM327** - убедиться что адаптер работает с Torque
3. **Создать структуру проекта** - следовать описанной архитектуре
4. **Разработка поэтапно** - не пропускать фазы
5. **Тестирование на каждом этапе** - не накапливать баги
6. **Фиксировать проблемы** - вести лог известных issues

**Готовы начать?** Предлагаю начать с создания структуры Flutter проекта и настройки зависимостей!
