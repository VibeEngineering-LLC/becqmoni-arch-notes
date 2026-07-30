# BecqMoni — карта связей модулей и данных

Дерево: `<клон BecqMoni>` (ветка fix/roi-wizard-defects), проект
`BecquerelMonitor` (C#/WinForms). Все пути ниже — от корня дерева; строки — по состоянию
рабочей копии на 30.07.2026. Режим обследования: read-only, только чтение кода.

Примечание о формате документа спектра: в задании упоминалось расширение «.bmn» —
в этом дереве его нет. Документ спектра сериализуется в **.xml**
(`BecquerelMonitor/DocumentManager.cs:23-29` — `Extension => ".xml"`, фильтр диалогов
`Properties/Resources.resx`: `Spectrum Files (*.xml)`).

---

## 1. Реестр сущностей данных

### 1.1 DeviceConfigInfo — конфигурация прибора
- Класс: `BecquerelMonitor/DeviceConfigInfo.cs:7`.
- Сериализация: XML-файл на конфиг в `%AppData%\BecqMoni\config\device\*.xml`
  (пути — `BecquerelMonitor/Package.cs:60-82`; чтение — `DeviceConfigManager.cs:70-106`,
  запись — `DeviceConfigManager.cs:229-241`, атомарно через `Utils.AtomicFileWriter`).
- Ключевые поля:
  - `FormatVersion` — строка `"120920"` (`DeviceConfigInfo.cs:473`, `InitFormatVersion` :361-364).
    Иное значение = старый формат → повторная десериализация как `DeviceConfigInfo_097b`
    и миграция (`DeviceConfigManager.cs:82-89`).
  - `Guid` — строковый GUID, первичный ключ конфига (ключ `DeviceConfigMap`).
  - `Name` — отображаемое имя (уходит в `DeviceConfigReference.Name`).
  - `DeviceType` — **строковый идентификатор** типа прибора: `"AudioInputDevice"`,
    `"AtomSpectraVCP"`, `"RadiaCode"`, `"Obsidian"` (`DeviceConfigInfo.cs:506`), ключ в
    `DeviceType.DeviceTypeMap` (`DeviceType.cs:31-70`).
  - `EnergyCalibration` (Polynomial/Nonlinear), `StabilizerConfig`, `PeakDetectionMethodConfig`
    (FWHM + деконволюция), `InputDeviceConfig` (полиморфный по XmlElement-атрибутам :209-224).
  - `DoseRateConfig` → список `DoseRateCalibrationPoint` (см. 1.8).
  - `BackgroundSpectrumPathname` — **абсолютный путь** к файлу фонового спектра (:310-320).
  - `EfficencyROIGuid` (:322-332) — строковый GUID ROI-конфига, назначенного прибору как
    кривая эффективности. Nullable; отсутствует в старых файлах.

### 1.2 ROIConfigData — конфигурация ROI (двойного назначения: области + кривая эффективности)
- Класс: `BecquerelMonitor/ROIConfigData.cs:8`.
- Сериализация: XML-файл на конфиг в `%AppData%\BecqMoni\config\ROI\*.xml`
  (`ROIConfigManager.cs:70-110` чтение, :247-260 запись).
- Ключевые поля:
  - `FormatVersion` `"120920"` (:226), `Guid` (ключ `ROIConfigMap`), `Name`, `LastUpdated`
    (по нему сортировка списка, `IComparable` :213-217 — убывание даты).
  - `ROIDefinitions` — список областей (`ROIDefinitionData`).
  - `ROIEfficiency` — список точек кривой эффективности (`ROIEfficiencyData`: Energy,
    Efficiency, ErrorPercent — `ROIEfficiencyData.cs:7-24`).
  - `HasEfficiency` — вычисляемое: `roiEfficiency.Count > 1` (:130-136), т.е. **кривой
    считается набор минимум из двух точек**; одна точка молча означает «кривой нет».

### 1.3 ROIDefinitionData — одна область ROI
- Класс: `BecquerelMonitor/ROIDefinitionData.cs:7`.
- Сериализуется внутри ROIConfigData; runtime-поля результатов помечены `[XmlIgnore]`.
- Поля: `Name` (по нему же ссылаются ROIReferenceData — см. 2.7), `PeakEnergy`,
  `LowerLimit`/`UpperLimit` (кэВ), `Color`, `HalfLife` (**в годах**; 0 = поправка на распад
  не применяется), `Intencity` (%), **`BecquerelCoefficient` / `BecquerelCoefficientError`**
  (:102-127) — «коэффициент активности» K: A[Бк] = (N/t)·K, `ROIPrimitives`
  (полиморфный список: `ROISimpleDifferenceData`, `ROICovellMethodData`, `ROIReferenceData`
  — XmlArrayItem :174-177). Runtime: `IsValidResult`, `ResultCount`, `ResultError`, `MDA`
  (:208-267) — кэш последнего расчёта, живёт прямо в конфиге-объекте.

### 1.4 ROIPrimitiveData и наследники
- База: `BecquerelMonitor/ROIPrimitiveData.cs:6`; в XML — **строковые** `PrimitiveType`
  и `OperationType`; резолвятся при загрузке через словари
  `ROIPrimitiveDefinition.DefinitionsMap` / `ROIPrimitiveOperation.OperationsMap`
  (`ROIConfigManager.cs:90-91`). Имена-ключи: `"BG difference"`, `"Covell Method"`,
  `"Reference"` (`ROIPrimitiveDefinition.cs:120-126`); операции `"Addition"`/`"Subtraction"`
  (`MeasurementResultManager.cs:443-446`).
- `ROIReferenceData.Reference` (`ROIReferenceData.cs:9-19`) — **строковое имя** другой
  области того же ROI-конфига.
- `Coefficient`/`CoefficientError` — множитель вклада примитива (по умолчанию 1.0,
  `ROIPrimitiveData.cs:171`).

### 1.5 NuclideDefinition / NuclideDefinitionFile / NuclideSet — библиотека нуклидов
- Классы: `BecquerelMonitor/NuclideDefinition.cs:9`, `NuclideDefinitionFile.cs:7`,
  `NuclideSet.cs:5`.
- Сериализация: **один файл** `%AppData%\BecqMoni\config\NuclideDefinition.xml`
  (`Package.cs:108-118`; менеджер `NuclideDefinitionManager.cs:82-127`, атомарная запись,
  событие `NuclideDefinitionListChanged` после записи :58, :122-125).
- `NuclideDefinition`: `Name` (несёт конвенцию цепочки в скобках — см. 5.1), `Energy` (кэВ),
  `HalfLife` (годы; 0 у ХРИ/вторичных/долгоживущих), `Intencity` (% на распад корня цепочки
  после пересчёта; 0 = маркер — см. 5.2), `Visible`, `NuclideColor`,
  **`Sets` — `HashSet<Guid>`** (:136-146) — членство записи в наборах,
  **`IsAnchor`** (:152-162) — якорная линия для библиотечного фита (в старых файлах
  отсутствует → false).
- `NuclideSet`: `Id` (**Guid**, а не строка), `Name`, `HideUnknownPeaks`.
- `NuclideDefinitionFile`: два списка — `NuclideDefinitions` + `NuclideSets`.

### 1.6 Документ спектра: ResultDataFile / ResultData / EnergySpectrum
- `ResultDataFile` (`BecquerelMonitor/ResultDataFile.cs:6`): `FormatVersion "120920"` (:59) +
  `List<ResultData>`. Сериализуется в пользовательский .xml целиком
  (`DocumentManager.SaveDocument` :1283-1315, атомарно), читается
  `DocumentManager.OpenDocument` :279-366.
- `ResultData` (`BecquerelMonitor/ResultData.cs:8`) — один спектр в документе.
  Сериализуемое: `SampleInfo`, **`DeviceConfigReference`** (:74-84),
  **`ROIConfigReference`** (:99-109), `BackgroundSpectrumFile` (**только имя файла**,
  инвалидные символы вычищаются :111-129), `StartTime`/`EndTime`, `PresetTime`,
  `EnergySpectrum`, `BackgroundEnergySpectrum`, `PulseCollection`, `FwhmCalibration`
  (:332-334).
  `[XmlIgnore]` (живые связи, восстанавливаются при открытии): **`DeviceConfig`** (:61-72),
  **`ROIConfig`** (:86-97), `BackgroundSpectrumPathname` (:131-142), `MeasurementController`,
  `MeasurementResultCollection`, `DetectedPeaks`, `PeakDetectionMethodConfig`,
  `ResultDataStatus`. Дефолты полей — **непустые объекты** (`deviceConfig = new
  DeviceConfigInfo()` :422, `roiConfig = new ROIConfigData()` :426) — при не-найденном Guid
  спектр молча живёт с пустым конфигом (см. 5.5).
- `DeviceConfigReference` / `ROIConfigReference` (`DeviceConfigReference.cs:4`,
  `ROIConfigReference.cs:4`): пара `Name` + `Guid` (строки). Создаются
  `CreateReference()` (`DeviceConfigInfo.cs:450-457`, `ROIConfigData.cs:203-210`).
- `EnergySpectrum` (`BecquerelMonitor/EnergySpectrum.cs:6`): `NumberOfChannels`,
  `ChannelPitch`, `EnergyCalibration` (свой экземпляр, клон приборной на момент создания),
  `TotalPulseCount`/`ValidPulseCount`, `MeasurementTime`/`LiveTime`, `SerialNumber`,
  `Spectrum int[]` (XmlArrayItem "DataPoint" :117). `Increment(value)` :172-179 —
  канал = высота импульса / ChannelPitch.

### 1.7 Результат измерения (runtime, не сериализуется)
- `MeasurementResultCollection` (`MeasurementResultCollection.cs:6`): ссылки на
  `ResultData`, `ROIConfig`, `MeasurementTime`, `List<MeasurementResult>`.
- `MeasurementResult` (`MeasurementResult.cs:4`): ссылка на `ROIDefinitionData` +
  `ResultValue`/`ResultError`/`MDA`/`IsValid`.
- Производитель: `MeasurementResultManager.Calculate` (`MeasurementResultManager.cs:136-201`)
  и `CalculateROI` (:204-437); конверсия единиц — `Translate` (:10-98), поправка на распад —
  `Correct` (:101-133). Результат кладётся в `ResultData.MeasurementResultCollection`
  (`MainForm.cs:620`).

### 1.8 Дозовые структуры
- `DoseRateConfig` (`DoseRateConfig.cs:7`) — список `DoseRateCalibrationPoint`, сериализуется
  внутри device-XML.
- `DoseRateCalibrationPoint` (`DoseRateCalibrationPoint.cs:5`): `LowerBound`/`UpperBound`
  (кэВ), `CPS`, `EtalonDoseRateValue`; `Sensitivity` — **вычисляемое** в сеттерах CPS и
  Etalon (:44-52, :62-72), read-only свойство (:75-81) → в XML не пишется, восстанавливается
  при десериализации порядком присвоений.
- `DoseRate` (`DoseRate.cs`) — пара Rate/Error, результат `DoseRateManager.Calculate`
  (`DoseRateManager.cs:20-81`).

### 1.9 LibraryPeakFitter — вход/выход
- Класс: `BecquerelMonitor/LibraryPeakFitter.cs:30`. Статический `Fit(...)` :457-466:
  вход — `EnergySpectrum` (+фон), провайдер SNIP-континуума, `FwhmCalibration`,
  найденные `List<Peak>`, `NuclideSet`, **снимок** `List<NuclideDefinition>`,
  `FWHMPeakDetectionMethodConfig`, опционально `ROIConfigData efficiencyConfig`
  (кривая прибора).
- Выход — `LibraryFitResult` (:421-428): `AddedPeaks` (`LibraryCandidate`: Nuclide, Channel,
  Fwhm, Amplitude, Area, Z :404-419), `ReplacedPeaks` (центроиды блендов на удаление),
  `AnchorPeaks` (пики, включившие фит).
- Внутренняя `EfficiencyShape` (:302-348) оборачивает `ROIAriphmetics` над
  `ROIConfigData.ROIEfficiency`; вне покрытия узлов — NaN, экстраполяции нет.
- Гейты-переключатели: `UseDevianceGate=false` :101, `UseBackgroundShapeGate=false` :128,
  `UseChainConsistencyVeto=true` :180 (`ChainScatterLimit=1.25` :185,
  `ChainConsistencyMinLines=6` :202), `UseChainVetoFallback=true` :217,
  `UseAbsenceVeto=true` :235, `UseOutlierTrim=false` :275.

### 1.10 Peak (runtime)
- `BecquerelMonitor/Peak.cs:10`: Energy, SNR, FWHM, Channel, **`Nuclide`** (ссылка на
  `NuclideDefinition`), `PeakSearchOrigin` (FWHMPeakFinder / RJMCMC / Library),
  `DeconvolutionInfo`, `IsLibraryAnchor`. Живёт в `ResultData.DetectedPeaks`.

### 1.11 GlobalConfigInfo
- `BecquerelMonitor/GlobalConfigInfo.cs:6` — окна, язык, `MeasurementConfig`
  (DetectionLevel, ErrorLevel, ShowValuesForNDResult), `ChartViewConfig`, автосохранение.
- Файл `%AppData%\BecqMoni\config\BecquerelMonitor.xml`
  (`GlobalConfigManager.cs:42-47, 76-81`, путь `Package.cs:84-94`).

---

## 2. Матрица связей

| # | Откуда → Куда | Механизм | Кто пишет | Кто читает |
|---|---|---|---|---|
| 1 | ResultData.DeviceConfigReference → DeviceConfigInfo | **Guid-строка** | `DCControlPanel.cs:452` (выбор в панели), `DocEnergySpectrum.cs:414` (новый документ) | `DocumentManager.PrepareDeviceConfig` :1355-1372 — линейный перебор `DeviceConfigList`, совпадение по `Guid`; не найдено → **молча** остаётся дефолтный конфиг |
| 2 | ResultData.ROIConfigReference → ROIConfigData | **Guid-строка** | `DCControlPanel.cs:483`, `DocEnergySpectrum.cs:459`, `DocumentManager.SplitDocEnergySpectrum` :1445 | `DocumentManager.PrepareROIConfig` :1375-1390 (то же: тихий не-матч). Поле `Name` ссылки **никем не читается** (grep по дереву пуст) — чисто информационное |
| 3 | DeviceConfigInfo.EfficencyROIGuid → ROIConfigData | **Guid-строка** | `DeviceConfigForm.cs:1715` (сброс), :1724 (выбор через словарь index→guid `effROIdic` :620); `SelectROIDialog.cs:111` (галочка «привязать к прибору») | `DocEnergySpectrum.CreateResultData` :451-457 (кладёт кривую в `ResultData.ROIConfig` нового спектра), `DCPeakDetectionView.BoundEfficiencyConfig` :49-58 (в фиттер — только при `ROIConfig.Guid == EfficencyROIGuid`), `DeviceConfigForm.cs:610-612`, `SelectROIDialog.cs:50-53` |
| 4 | ResultData → фоновый спектр | **путь к файлу**: сериализуется только имя (`BackgroundSpectrumFile`), полный путь живёт в `[XmlIgnore] BackgroundSpectrumPathname` и в `DeviceConfigInfo.BackgroundSpectrumPathname` (абсолютный) | `DCControlPanel.cs:509-510`, `DocumentManager.CreateDocument` :99-100 (путь берётся из конфига прибора) | `DocumentManager.LoadBackgroundSpectrum` :1393-1436 — читает файл как ResultDataFile и берёт `ResultDataList[0].EnergySpectrum` |
| 5 | NuclideDefinition.Sets → NuclideSet.Id | **HashSet\<Guid\>** (настоящий Guid) | `NuclideSetForm.cs:200-204` (галочки членства), `RoiWizard/SetExporter.cs:213` (`definition.Sets.Add(set.Id)`), `SetExporter.MergeIntoLibrary` :246 | `PeakDetector.MatchNuclide` :365 (`Sets.Contains(nuclideSet.Id)`), `LibraryPeakFitter.Fit` :480 |
| 6 | Линия → цепочка распада | **конвенция в строке Name**: текст в ПОСЛЕДНИХ скобках = имя корня («Bi-214 (Ra-226)» → «Ra-226»), без скобок — имя целиком | `NucBase/NucBase.cs:501-504` (импорт ряда: `name += " (корень)"`), `RoiWizard/SpectralLine.LibraryName` :69-105 (та же конвенция + спец-обработка слитых линий и суффиксов «X L») | `LibraryPeakFitter.ChainOf` :952-962 → группировка bound-групп :604; `RoiWizard/SetChecker.ChainOf` :279 |
| 7 | ROIReferenceData.Reference → другая область | **строковое имя** области в том же конфиге | `ROIReferenceControl` (форма ROI) | `MeasurementResultManager.CalculateROI` :369-388 — поиск `roidefinitionData.Name == reference`, рекурсия с лимитом 10 :204-210 |
| 8 | ROIPrimitiveData → алгоритм расчёта | **строковые** PrimitiveType/OperationType | ROI-формы; `SetExporter.AddDifferencePrimitive` :96-111 (поиск по имени `"BG difference"`/`"Addition"`, намеренно не по индексу — комментарий :78-80) | `ROIConfigManager.LoadAllConfigFiles` :90-91 (резолв в объекты через `DefinitionsMap`/`OperationsMap`), `MeasurementResultManager` :443-446 |
| 9 | DeviceConfigInfo.DeviceType → контроллер прибора | **строка-Id** → `DeviceType.DeviceTypeMap` → `Type` контроллера | конструкторы/миграции (`DeviceConfigInfo.cs:423,506`) | `MeasurementController.CreateDeviceController` :215-260 (`Activator.CreateInstance`), `DeviceConfigForm.cs:432` |
| 10 | Фиттер → кривая эффективности | передача `ROIConfigData` параметром; гейт по Guid-равенству | `DCPeakDetectionView.cs:217` (снапшот: `ROIConfig = BoundEfficiencyConfig(...)` — глубокая копия) | `PeakDetector.AppendLibraryPeaks` :113-128 → `LibraryPeakFitter.Fit(..., resultData.ROIConfig)`; `EfficiencyShape.From` :313-329 → `ROIAriphmetics.CalculateEfficiency` (`Utils/ROIAriphmetics.cs:26-42`, монотонный кубический сплайн :175-176) |
| 11 | Дозовая калибровка: форма ↔ конфиг | список объектов ↔ строки таблицы | загрузка `DeviceConfigForm.cs:530-540`; сохранение :852-861 (таблица → новый список точек) | `MainForm.ShowDoseRate` :634-647 → `DoseRateManager.Calculate` :20-81 |
| 12 | ROI-область → коэффициент активности | число `BecquerelCoefficient`, **вычисляется при создании области** и замораживается | `ROIConfigForm.button9_Click` :951-962: `K = (1/ε(E)) / (I/100)`, ошибка = K·(δε/100) :957-961; вручную — :888-899 | `MeasurementResultManager.Translate` :44-56: `Bq = cps·K`, ошибка по квадратуре :52 |
| 13 | Активность по выделению (без ROI) | on-the-fly из кривой + Intencity пика | — | `EnergySpectrumView.cs:736-763`: ровно один пик в выделении, `Nuclide.Intencity > 0`, `roiConfig.HasEfficiency` → тот же `K = (1/ε)/(I/100)` :742 |
| 14 | Панель управления → конфиги | **индекс combobox = индекс списка менеджера** | выбор пользователя | `DCControlPanel.cs:451` (`DeviceConfigList[SelectedIndex]`), :479 (`ROIConfigList[SelectedIndex]`); заполнение имён :76-93, обратный поиск по Guid для подсветки :284-312 |
| 15 | SelectROIDialog → выбор кривой | **строковое имя** из combobox → поиск `Name ==` | — | `SelectROIDialog.cs:103-107`: коллизия имён ROI-конфигов отдаёт первый попавшийся |
| 16 | RoiWizard → библиотека и ROI | объекты через менеджеры (файлы не пишет сам) | `RoiWizardForm.cs:1949-1973` (`CreateConfig`/`SaveConfig`), :2067-2074 (`MergeIntoLibrary` + `SaveDefinitionFile`) | далее штатные читатели менеджеров |
| 17 | NucBase (nucdb.sqlite) → библиотека | чтение SQLite (read-only, `NucBase/DataBase.cs:18-19`) → `NuclideDefinition` | `NucBase.cs:438-527` (импорт с пересчётом интенсивности на распад корня :448-494) | далее как 1.5 |
| 18 | LSRM-файл эффективности → ROI-конфиг | парсинг TSV → `ROIEfficiency` | `ROIConfigManager.ImportEffCalcMCtoROI` :290-350 (создаёт ROI-файл, состоящий из одной кривой; вызов `MainForm.cs:2545`), `DeviceConfigForm.buttonLoadEff_Click` :2436-2491 (для оценки дозовой калибровки) | читатели кривой (связи 10, 12, 13) |

---

## 3. Формы (WinForms) и их роли

- **MainForm** (`MainForm.cs`) — хаб: таймер 100 мс; каждые ~500 мс
  `ShowMeasurementResult` :612-631 (пересчёт активностей активного документа →
  `ResultData.MeasurementResultCollection` :620 и раздача в DCResultView) и `ShowDoseRate`
  :634-647; первый запуск — копирование поставочного `config\` в `%AppData%\BecqMoni`
  :102-117; инициализация словарей типов :148-151.
- **DCControlPanel** (`DCControlPanel.cs`) — выбор Device/ROI-конфига активного спектра
  (пишет `DeviceConfig(+Reference)` :451-452 и `ROIConfig(+Reference)` :479-483 по индексу
  списка), назначение/сброс фонового спектра :507-541, старт/стоп измерения.
- **DeviceConfigForm** (`DeviceConfigForm.cs`) — редактор DeviceConfigInfo: калибровка
  энергии, FWHM/деконволюция, стабилизатор (tableModel3 :843-849), дозовая таблица
  (tableModel4 :530-540/:852-861) с оценкой по эталонному спектру и LSRM-кривой
  :2493-2577, привязка кривой эффективности (`selectEffROI` + словарь index→Guid :615-626,
  :1720-1727). Сохранение — `DeviceConfigManager.SaveConfig`.
- **ROIConfigForm** (`ROIConfigForm.cs`) — редактор ROIConfigData: области, примитивы,
  создание области из нуклида с автоподсчётом `BecquerelCoefficient` :936-968; кнопка
  кривой → **ROIEditEfficiencyDialog** (ручной ввод точек `ROIEfficiency`,
  `ROIEditEfficiencyDialog.cs:39-114`).
- **NuclideDefinitionForm** (`NuclideDefinitionForm.cs`) — CRUD записей библиотеки; каждая
  правка завершается `SaveDefinitionFile` (:101, :129, :248).
- **NuclideSetForm** (`NuclideSetForm.cs`) — наборы: создаёт `NuclideSet` (Guid), правит
  членство `def.Sets` :200-204; правит и сортирует **живой общий список** — из-за этого
  детекция работает по снимкам (`PeakDetector.cs:13-19`, `DCPeakDetectionView.cs:222-227`).
- **DCPeakDetectionView** (`DCPeakDetectionView.cs`) — запуск детекции пиков: снапшот
  ResultData на UI-потоке :189-218 (в т.ч. `ROIConfig = BoundEfficiencyConfig` :217),
  `Task.Run(DetectPeak)` :230, результат → `ResultData.DetectedPeaks` :235.
- **DCResultView** (`DCResultView.cs`) — таблица активностей: получает
  `MeasurementResultCollection`, гоняет через `Translate`(единицы)/`Correct`(распад)
  :100-105, сравнивает конфиги по `ROIConfig.Guid` :91.
- **DCSampleInfoView** (`DCSampleInfoView.cs:76-93`) — пишет `SampleInfo`
  (Name/Location/Time/Weight/Volume — знаменатели Бк/кг и Бк/л).
- **EnergySpectrumView** (`EnergySpectrumView.cs`) — отрисовка спектра; берёт
  `roiConfig = activeResultData.ROIConfig` :1164; штрихи-маркеры областей по `Intencity`
  (`ShowROIReferencePeak` :2801-2870); аналитика выделения с активностью по кривой
  :703-763.
- **SelectROIDialog** (`SelectROIDialog.cs`) — выбор кривой эффективности для спектра;
  умеет записать выбор в конфиг прибора (:108-113).
- **RoiWizardForm** (`RoiWizard/RoiWizardForm.cs`) — конструктор наборов/ROI из каталога
  nucdb.sqlite; наружу отдаёт объекты через `ROIConfigManager`/`NuclideDefinitionManager`
  (:1949-1973, :2067-2074), сам файлов не пишет.
- **NucBase** (`NucBase/NucBase.cs`) — браузер ядерной базы; импорт линий/рядов в
  библиотеку :438-527.
- **DCEnergyCalibrationView / DCFwhmCalibrationView / DCPulseView / DCCountRateView /
  DCSpectrumListView** — вспомогательные панели (калибровка по точкам, FWHM, импульсы,
  скорость счёта, список спектров).

---

## 4. Потоки данных

### 4а. Спектр от прибора → калибровка → отображение
1. Старт: `MeasurementController.StartRecording` → `CreateDeviceController`
   (`MeasurementController.cs:215-260`): по строке `DeviceConfig.DeviceType` через
   `DeviceType.DeviceTypeMap` создаётся контроллер (`Activator.CreateInstance`).
2. Аудио-тракт: `AudioInputDeviceController.StartMeasurement`
   (`AudioInputDeviceController.cs:28-74`) отдаёт детектору импульсов ссылку
   `pulseDetector.EnergySpectrum = resultData.EnergySpectrum` (:74).
3. Каждый импульс: `CorrelativePulseDetector.ProcessPulse`
   (`CorrelativePulseDetector.cs:104-112`) → `EnergySpectrum.Increment(height)`
   (`EnergySpectrum.cs:172-179`): канал = высота/`ChannelPitch`, `Spectrum[ch]++`.
4. Калибровка: у спектра **своя** `EnergyCalibration` — клон приборной на момент создания
   ResultData (`DocEnergySpectrum.cs:416`); дальше живёт независимо (правится в
   DCEnergyCalibrationView / тулстрипе).
5. Отображение: `DocEnergySpectrum.UpdateEnergySpectrum` :478-483 →
   `EnergySpectrumView` (каналы→кэВ через `ChannelToEnergy`, поканальная отрисовка,
   сглаживание из `GlobalConfig.ChartViewConfig`).
6. При открытии файла обратная сборка связей: `DocumentManager.OpenDocument` :279-366 →
   `PrepareDeviceConfig`/`PrepareROIConfig` (Guid-матчи), досчёт TotalPulseCount :347-351,
   `ResultDataList[0].Selected = true` :363.

### 4б. Активность по ROI — откуда каждая величина A = N/(t·ε·I)
Формула в коде разложена на два шага; ε и I «сидят» в заранее посчитанном коэффициенте K:

1. **N (счёт)**: `MeasurementResultManager.CalculateROI` (`MeasurementResultManager.cs:204-437`)
   — по примитивам области: `ROISimpleDifferenceData` — сумма каналов
   [EnergyToChannel(Lower)..(Upper)] :234-261, фон приводится множителем
   `fgTime/bgTime` :270-275, при разных калибровках фон пересчитывается канал-в-канал
   через энергию :251-254; `ROICovellMethodData` — трапеция по боковым окнам :279-367;
   `ROIReferenceData` — счёт другой области по имени :369-391. Вклады примитивов
   складываются/вычитаются с коэффициентами :393-421.
2. **t**: `MeasurementTime` активного спектра (:157, знаменатель в `Translate` :46).
3. **ε и I**: `K = BecquerelCoefficient` области. Пишется при создании области из нуклида:
   `ROIConfigForm.cs:957` — `K = (1/effData.Efficiency) / (nuclideDefinition.Intencity/100)`,
   где ε — интерполяция `ROIAriphmetics.CalculateEfficiency` по `ROIEfficiency` конфига
   (`Utils/ROIAriphmetics.cs:26-42`), I — из библиотеки нуклидов. Либо вручную
   (`ROIConfigForm.cs:888`). **После записи K живёт своей жизнью** — смена кривой или
   интенсивности в библиотеке на него не влияет.
4. **A**: `Translate` :48 — `resultBq = (N/t) · K`; ошибка :52 (квадратура счётной и
   коэффициентной), MDA :55-56; Бк/кг и Бк/л — деление на `SampleInfo.Weight/Volume`
   :74-91. Поправка на распад — `Correct` :101-133 (нужны `SampleInfo.Time`, `EndTime`,
   `HalfLife` области; HalfLife=0 → пропуск :126).
5. MDA внутри CalculateROI — форма Карри :426-435.
6. Периодичность: `MainForm.ShowMeasurementResult` :612-631 (по таймеру) → DCResultView.

Альтернативная ветка «по выделению»: `EnergySpectrumView.cs:736-763` — ε и I берутся
на лету (кривая из `ResultData.ROIConfig`, I из `Peak.Nuclide.Intencity`), K не
замораживается.

### 4в. Идентификация: библиотека → LibraryPeakFitter → вердикты
1. Библиотека и набор: `NuclideDefinition.xml` → `NuclideDefinitionManager`;
   выбранный `NuclideSet` — в `DCPeakDetectionView.selectedNuclideSet`.
2. Снапшоты на UI-потоке: `DCPeakDetectionView.UpdatePeakDetectionResult` :189-234
   (спектр, фон, конфиг, FWHM-калибровка, список нуклидов копией :226-227, кривая
   эффективности `BoundEfficiencyConfig` :217).
3. `PeakDetector.DetectPeak` (`PeakDetector.cs:11-94`): FWHM-finder :392-427 →
   `CollectPeaks` :217-251 (пик → `MatchNuclide` :358-376: ближайший по
   `|ΔE|/E < Tolerance/100` из видимых линий набора) → опц. RJMCMC-деконволюция :58-79 →
   `AppendLibraryPeaks` :100-167.
4. `LibraryPeakFitter.Fit` (`LibraryPeakFitter.cs:457` и далее):
   - гейт якоря: линия с `IsAnchor` должна совпасть с найденным пиком в 0.5·FWHM
     :498-529 (сдвиг калибровки — с сильнейшего якоря);
   - сайты всех линий набора :539-563 (`Chain = ChainOf(line)` :561);
   - bound-группы: кластеризация ближе 0.85·FWHM, в группу — только линии одной цепочки
     с `Intensity > 0`, веса ∝ Intencity :598-616;
   - Пуассон-фит амплитуд; значимость Fisher z ≥ 4 (:35, :2169-2186);
   - вето по согласованности набора (кривая S/I vs E, `ChainConsistent` :2016-2096;
     при наличии внешней кривой — фикс. форма, свободен только масштаб :2042-2066),
     вето по отсутствиям, запасной shape-гейт — переключатели :101-275;
   - выход: `AddedPeaks` (origin=Library, `PeakDetector.cs:160`), `ReplacedPeaks`
     (удаляются из списка :140-143), `AnchorPeaks` (флаг `IsLibraryAnchor` :133-136).
5. Результат в `ResultData.DetectedPeaks` (`DCPeakDetectionView.cs:235`) → таблица пиков и
   отрисовка.

### 4г. Доза: калибровка → показание
1. Калибровка (ручная): `DeviceConfigForm` таблица 4 — точки (LowerBound, UpperBound, CPS,
   EtalonDoseRateValue); сохранение в `DoseRateConfig.DoseRateCalibrationPoints`
   :852-861 → device-XML.
2. Калибровка (оценка): эталонный спектр (:2411-2434, **`ResultDataList[0]` без проверки**
   :2430) + LSRM-кривая ε (:2436-2491) + ожидаемая МД →
   `CalculateDoseRateConfig` :2522-2577: 16 фиксированных энергоинтервалов 40–3000 кэВ,
   зашитые таблицы μ(E) и R→Sv :2524-2526; на выходе точки с `CPS=1` и
   `EtalonDoseRateValue = coeff·μ·(R→Sv)·E/ε` :2564-2574.
3. Показание: `MainForm.ShowDoseRate` :634-647 (каждый тик при наличии точек) →
   `DoseRateManager.Calculate` (`DoseRateManager.cs:20-81`): по каждой точке счёт в
   [EnergyToChannel(Lower)..(Upper)] × `point.Sensitivity`, сумма / `MeasurementTime`
   :57-79; `Sensitivity = Etalon/CPS` — вычислено сеттерами точки
   (`DoseRateCalibrationPoint.cs:44-52`). Вывод — строка статуса.

---

## 5. Неявные связи и конвенции

1. **Цепочка распада в скобках имени.** `LibraryPeakFitter.ChainOf`
   (`LibraryPeakFitter.cs:949-962`): «идентификатор цепочки — текст в последних скобках
   имени („Bi-214 (Ra-226)“ → „Ra-226“), иначе имя целиком». Писатели конвенции:
   `NucBase/NucBase.cs:503` (`formattedName += " (" + FormatIsotopeName(...) + ")"`),
   `RoiWizard/SpectralLine.cs:69-105` (LibraryName выносит интервал слитой линии ИЗ скобок
   :74-91 и дописывает цепочку линиям с суффиксами :98-103 — иначе «U-238 X L» стал бы
   собственной цепочкой). Следствие: **переименование нуклида ломает BR-связку молча**,
   а любые скобки в конце имени читаются как цепочка.
2. **Intencity — поле с тремя смыслами.** (а) В `NuclideDefinition` — интенсивность
   в % **на распад корня цепочки** (пересчёт при импорте ряда, `NucBase.cs:448-494`;
   без маркера цепочки — на распад самого нуклида). (б) **`Intencity == 0` — маркер**:
   запись (ХРИ, вторичный пик) не попадает в BR-связку (`LibraryPeakFitter.cs:607`
   `members.All(m => m.Intensity > 0.0)`) и не даёт активности по выделению
   (`EnergySpectrumView.cs:736`); конвенция задокументирована в
   `RoiWizard/SetExporter.cs:198-212`. (в) В `ROIDefinitionData` — копия для отрисовки
   штрихов (`EnergySpectrumView.ShowROIReferencePeak` :2810-2833; условие :2812
   `(Intencity < 100.0 || Intencity > 0.0)` — тавтология, истинно для любого числа).
3. **FormatVersion "120920"** — магическая строка-константа в трёх сущностях
   (`DeviceConfigInfo.cs:473`, `ROIConfigData.cs:226`, `ResultDataFile.cs:59`);
   несовпадение у device-конфига включает миграцию из формата 0.97b
   (`DeviceConfigManager.cs:82-89`), у ROI — просто перештамповку
   (`ROIConfigManager.cs:82-85`).
4. **Fallback на [0].** Новый спектр получает `DeviceConfigList[0]`
   (`DocEnergySpectrum.cs:413`) и — при отсутствии `EfficencyROIGuid` —
   `ROIConfigList[0]` (:457). Списки отсортированы по `LastUpdated` убыванием
   (`ROIConfigData.cs:213-217`, `DeviceConfigInfo.cs:460-464`), т.е. «первый» = последний
   сохранённый, и **состав по умолчанию меняется от любого сохранения любого конфига**.
   `ResultData.ROIConfig` при этом — поле двойного назначения: «ROI для таблицы» и
   «кривая эффективности»; фиттер защищается от чужого конфига проверкой Guid
   (`DCPeakDetectionView.cs:49-58`, комментарий :210-216).
5. **Тихий не-матч Guid при открытии документа.** `PrepareDeviceConfig`/`PrepareROIConfig`
   (`DocumentManager.cs:1355-1390`) при отсутствии Guid в списках ничего не сообщают;
   спектр остаётся с дефолтными **непустыми** объектами (`ResultData.cs:422,426`) —
   внешне это выглядит как «конфиг есть, но пустой».
6. **ROIReferenceData.Reference — ссылка по имени области** внутри того же конфига
   (`MeasurementResultManager.cs:374` `if (roidefinitionData.Name == roireferenceData.Reference)`).
   Переименование области или дубль имени рвёт/двоит ссылку без диагностики; ненайденная
   ссылка даёт вклад 0 молча (цикл просто не находит).
7. **SelectROIDialog выбирает по имени** (`SelectROIDialog.cs:105`
   `ROIConfigDatas[i].Name == comboBox1.SelectedItem.ToString()`) — при совпадении имён
   двух ROI-конфигов вернётся первый по списку, хотя везде остальное — Guid.
8. **Связь «индекс combobox = индекс списка менеджера»** в панели управления
   (`DCControlPanel.cs:451, 479`) и словарь index→Guid в DeviceConfigForm
   (`effROIdic`, :620, :1723). Работает, пока список не перестроился между заполнением и
   выбором.
9. **`HasEfficiency` = `Count > 1`** (`ROIConfigData.cs:130-136`) — кривая из одной точки
   молча считается отсутствующей.
10. **HalfLife в годах, 0 = «вечный/не поправлять»**: `MeasurementResultManager.Correct`
    :120-126 (в поставочных ROI есть HalfLife=0 — комментарий :124-125);
    конвенция мастера: ≥1e9 лет → писать 0 (`SetExporter.cs:59`, :193-195 — у ХРИ и
    вторичных HalfLife не заполняется, «конвенция файла-образца BecqMoni»).
11. **`DoseRateCalibrationPoint.Sensitivity` — вычисляемое и несериализуемое** (read-only
    свойство): значение восстанавливается побочным эффектом сеттеров `CPS` и
    `EtalonDoseRateValue` при десериализации (`DoseRateCalibrationPoint.cs:35-81`).
12. **Строки-ключи алгоритмов**: примитивы `"BG difference"`, `"Covell Method"`,
    `"Reference"`; операции `"Addition"`, `"Subtraction"`
    (`ROIPrimitiveDefinition.cs:120-126`, `MeasurementResultManager.cs:443-446`);
    типы приборов `"AudioInputDevice"`, `"AtomSpectraVCP"`, `"RadiaCode"`, `"Obsidian"`
    (`DeviceType.cs:36-69`). Всё это — содержимое пользовательских XML: опечатка в файле
    = KeyNotFound при загрузке (`ROIConfigManager.cs:90-91` — в per-file try).
13. **Поля-близнецы ROI-запись ↔ NuclideDefinition.** При создании области из нуклида
    копируются Name, PeakEnergy, HalfLife, Intencity (`ROIConfigForm.cs:944-950`;
    `SetExporter.BuildRoiConfig` :51-60) и вычисляется K. Дальше копии живут независимо:
    правка библиотеки не трогает ROI-файлы, и наоборот. Аналогично «эталонная» связка
    имён: `ROIDefinitionData.Name` мастера = `SpectralLine.Label`, а
    `NuclideDefinition.Name` = `SpectralLine.LibraryName` — похожие, но разные строки.
14. **Дублирование между ROI-файлами.** Каждый поставочный файл `config/ROI/*.xml` несёт
    собственные копии одних и тех же областей (Cs-137, Ra-226, K-40 в «Atom Nano 3»,
    «Atom Nano 8», «Atom Spectra 2»...); кривая эффективности — тоже per-файл
    (`Obsidian Marinelli 0.5.xml` — 150 точек ROIEfficiencyData, большинство остальных — 0).
    Единственный общий справочник — `NuclideDefinition.xml`; ROI-файлы с ним не
    синхронизируются.
15. **Абсолютный путь фона в конфиге прибора** (`DeviceConfigInfo.BackgroundSpectrumPathname`)
    и «имя-файла-без-пути» в спектре (`ResultData.BackgroundSpectrumFile`) — две половины
    одной ссылки; при переносе на другую машину работает только та, где путь совпал.
16. **Runtime-кэш в конфиге**: `ROIDefinitionData.IsValidResult/ResultCount/ResultError/MDA`
    — результаты последнего расчёта живут в общем объекте конфига (тот же экземпляр у всех
    документов, `ROIConfigManager` отдаёт общий список), `MeasurementResultManager.Calculate`
    каждый раз сбрасывает флаги :163-166.
17. **Снимки против живых списков.** `NuclideDefinitions` правится и сортируется UI-формами
    in-place, поэтому детекция работает по копиям (`PeakDetector.cs:13-19`,
    `DCPeakDetectionView.cs:183-227`) — конвенция, а не механизм: любой новый потребитель
    обязан помнить о снимке сам.

---

## 6. Каталог файлов конфигурации

### 6.1 %AppData%\BecqMoni (не-standalone; standalone — те же имена рядом с exe)
Пути определяет `BecquerelMonitor/Package.cs:36-142`; при первом запуске поставочный
`config\` рядом с exe копируется в %AppData% целиком (`MainForm.cs:102-117`).

| Путь | Содержимое | Читает / пишет |
|---|---|---|
| `config\BecquerelMonitor.xml` | `GlobalConfigInfo` (окна, язык, MeasurementConfig, ChartViewConfig, автосейв) | `GlobalConfigManager` (`GlobalConfigManager.cs:42-47` / :76-81) |
| `config\device\*.xml` | по одному `DeviceConfigInfo` на файл (внутри: калибровка, стабилизатор, DoseRateConfig, EfficencyROIGuid, путь к фону) | `DeviceConfigManager` (:70-106 / :229-241); имя файла = `Filename`, ключ = `Guid`; дубль Guid → отказ с сообщением :92-95 |
| `config\ROI\*.xml` | по одному `ROIConfigData` на файл (области + ROIEfficiency) | `ROIConfigManager` (:70-110 / :247-260); дубль Guid → отказ :96-99 |
| `config\NuclideDefinition.xml` | `NuclideDefinitionFile`: вся библиотека нуклидов + все NuclideSets | `NuclideDefinitionManager` (:82-127); при отсутствии создаётся дефолт из 4 записей :130-153 |
| `config\layout\*.xml` | раскладки DockPanel (`ExpertMode.xml` в поставке) | `MainForm` (`MainForm.cs:3376`) |

### 6.2 Поставка (корень дерева)
- `config\` — образец, копируемый в %AppData% (см. выше): `config/ROI/` — 12 файлов
  (приборные наборы + `Ra-226 Intensities.xml`, `Th-232 Intensities.xml`),
  `config/device/` — 9 приборных конфигов, `config/NuclideDefinition.xml`,
  `config/layout/ExpertMode.xml`.
- `BecquerelMonitor\nucdb.sqlite` — ядерная база (decay_radiations, nuclides, decay_chain +
  добавленные families/xrf_elements/xrf_lines/chains/catalog_meta); открывается read-only
  из текущего каталога (`NucBase/DataBase.cs:18-19`); потребители — NucBase-формы и
  `RoiWizard/NuclideCatalog` (:1-21, singleton :54-60).
- `LSRM Geometries\` — модели и экспортированные кривые эффективности (формат импорта
  для `ImportEffCalcMCtoROI` / `buttonLoadEff_Click`).
- Документы спектров — пользовательские `.xml` (ResultDataFile), место хранения произвольное.

---

## 7. Не выяснено

1. **Расширение `.bmn`** в коде не встречается; документ спектра — `.xml`
   (`DocumentManager.cs:23-29`). Если `.bmn` существует в другой ветке/форке — в этом
   дереве следов нет.
2. `ROIConfigReference.Name` / `DeviceConfigReference.Name` — записываются в файл спектра,
   но ни одного читателя не найдено (grep по дереву); видимо, задел под «восстановление по
   имени», который не реализован.
3. Насколько `ResultData_097b`/`ResultDataFile_097b`/`ResultData_093b` ещё встречаются у
   пользователей — из кода не видно; ветки миграции живы (`DocumentManager.cs:446-520`).
4. Формат N42 (три схемы в `N42/`) прослежен только до `Util.ImportFromN42`; детальная
   карта полей N42 ↔ ResultData не строилась.
5. Точная семантика `EnergySpectrum.SerialNumber` (кто пишет при записи с прибора) — не
   прослежена по всем контроллерам (RadiaCode/Obsidian BLE-тракты не читались построчно).
6. `MeasurementResultCollection` нигде не сериализуется — сохранение результатов
   измерения в файл, по-видимому, происходит только косвенно (через кэш в
   ROIDefinitionData ничего в XML не уходит: поля XmlIgnore). Экспорт таблицы результатов
   в файл не искался.
