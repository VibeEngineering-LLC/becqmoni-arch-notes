# Карта меню BecqMoni (BecquerelMonitor)

**Источник:** клон `<клон BecqMoni>`, HEAD `d80c7ee` (локальная ветка `fix/roi-wizard-defects`,
линия roi-wizard-reworked). Построено только по коду:
`BecquerelMonitor\MainForm.Designer.cs` (структура MenuStrip), `BecquerelMonitor\MainForm.cs` (обработчики),
`BecquerelMonitor\MainForm.ru.resx` / `MainForm.resx` (подписи), `DocEnergySpectrum.Designer.cs`/`.cs`/`.resx`/`.ru.resx`
(тулбары и контекстное меню спектра), `DCPeakDetectionView.*`, `Utils\CalibrationGraph.*`, `Utils\FWHMCalibrationGraph.*`,
`ToolWindow.*`, `RoiWizard\RoiWizardForm.*`.

Подписи: **русская из ru.resx / английская из neutral resx**. Если русского ключа в ru.resx нет — пункт
и в русской локали показывается по-английски (WinForms fallback), это помечено «(RU-ключа нет)».
Все строки/номера указаны в файлах каталога `<клон BecqMoni>\BecquerelMonitor\`.

Главное меню MainForm: пять топ-пунктов (`MainForm.Designer.cs:113-118`). Собственного тулбара у MainForm **нет** —
только MenuStrip и StatusStrip; тулбары живут в окне спектра DocEnergySpectrum (см. раздел «Тулбары»).

---

## 1. Главное меню

### Файл / &File (`fileFToolStripMenuItem`)
При раскрытии пункты Сохранить/Закрыть/Объединение включаются по наличию активного документа — `MainForm.cs:1736`.

- **Новый / &New** — `Ctrl+N` — создаёт новый документ спектра (`documentManager.CreateDocument()`) и показывает его в док-панели — `MainForm.cs:920`
- **Открыть... / &Open...** — `Ctrl+O` — OpenFileDialog (мультивыбор), каждый файл открывается `documentManager.OpenDocument()` — `MainForm.cs:949`
- **Сохранить / &Save** — `Ctrl+S` — `SaveActiveDocument()` → `documentManager.SaveDocument()`; безымянный документ уходит в «Сохранить как» — `MainForm.cs:1670` (логика `MainForm.cs:1689,1703`)
- **Сохранить как / Save &As...** — `SaveDocumentWithName()` → `documentManager.SaveDocumentWithName()` — `MainForm.cs:1719`
- **Закрыть / &Close** — `Ctrl+W` — `CloseActiveDocument()`: подтверждение сохранения, остановка записи, закрытие документа — `MainForm.cs:1421` (логика `MainForm.cs:1552`)
- **О&бъединение спектров / Co&mbine Spectras** — открывает файл спектра и суммирует его с активным документом (`SpectrumAriphmetics.CombineWith`) — `MainForm.cs:1435`
- **Закрыть &Всё / Close &All** — цикл `CloseActiveDocument()` по всем открытым документам — `MainForm.cs:1426`
- ─────
- **Импорт / &Import** (подменю):
  - **Формат Atom Spectra v3 / Atom Spectra v3 format** — мультивыбор файлов, каждый импортируется в новый документ (`ImportDocumentAtomSpectra`) — `MainForm.cs:2727`
  - **Файл формата N42 / N42 file format** — импорт одного N42-файла в новый документ (`ImportDocumentN42`) — `MainForm.cs:2749`
  - **CSV (Channel,Counts) Файл / CSV (Channel,Counts) File** — импорт CSV канал-счёт в новый документ (`ImportCsvToDocument`, preset time берётся из панели управления) — `MainForm.cs:2610`
  - **LSRM EffCalcMC.txt -> ROI** (RU-ключа нет) — выбор файла EffCalcMC + диалог `EffCalcMCDialog` (имя ROI); `roiConfigManager.ImportEffCalcMCtoROI()` создаёт конфигурацию ROI из расчёта эффективности, обновляет список ROI в панели управления — `MainForm.cs:2529` ← **добавлено веткой roi-wizard**
  - **CSV (Energy,Counts) Файл / CSV (Energy,Counts) File** — импорт CSV энергия-счёт (`ImportCsvEnergyToDocument`) — `MainForm.cs:2628`
  - **SpectraLine GBS** (RU-ключа нет) — мультивыбор, импорт GBS-файлов SpectraLine (`ImportDocumentGBS`) — `MainForm.cs:2556`
  - **Импорт при помощи SpectUtils.. / Import using SpectUtils..** — мультивыбор, импорт через библиотеку SpecUtils (`ImportDocumentSpecUtils`) — `MainForm.cs:2582`
  - **Старый формат BecqMoni (v0.93Beta или ранее) / BecqMoni old format (v0.93Beta or earlier)** — `ImportDocument093b()` — `MainForm.cs:2702`
- **Экспорт / &Export** (подменю):
  - **Файл N42 / N42 file** — `ExportDocumentN42(activeDocument)` — `MainForm.cs:2767`
  - **Формат Atom Spectra v3 / Atom Spectra v3 format** — `ExportDocumentAtomSpectra(activeDocument)` — `MainForm.cs:2776`
  - **Файл &CSV / &CSV File** — `ExportDocumentToCsv(activeDocument)` — `MainForm.cs:1837`
  - **&Расширенный CSV файл / &Extended CSV File** — `ExportDocumentToECSV` с учётом текущего режима фона и сглаживания окна спектра — `MainForm.cs:1845`
  - **для FWHM / for FWHM** — **СКРЫТ В РАНТАЙМЕ** (`MainForm.cs:161` `Visible = false`) — пишет `fwhm.txt`: участок 627–780 кэВ с вычетом линейной подложки, опорные точки 530 и 780 кэВ (жёстко зашитый диапазон пика Cs-137; кодировка файла 932/Shift-JIS) — `MainForm.cs:3056`
- ─────
- **Выход / E&xit** — `Close()` главной формы — `MainForm.cs:752`

### Спектр / &Spectrum (`spectrumSToolStripMenuItem`)
Пункты действуют на активный спектр (ResultData) внутри активного документа. Доступность пересчитывается
при раскрытии: лимит спектров в файле, наличие фона, состояние записи — `MainForm.cs:1747`.

- **Новый / &New** — `AddNewSpectrum()`: добавляет новый спектр в активный документ, подтягивает фон из конфигурации устройства — `MainForm.cs:2871` (логика `MainForm.cs:2174`)
- **Удалить / &Delete** — `DeleteActiveSpectrum()`: удаляет активный спектр с подтверждением; последний спектр удалить нельзя — `MainForm.cs:2877` (логика `MainForm.cs:2251`)
- **Импорт из файла / &Import from File** — `LoadSpectrumFromFile()`: добавляет спектры из выбранных файлов в текущий документ — `MainForm.cs:2883` (логика `MainForm.cs:2339`)
- **&Export Spectrum to File** (RU-ключа `exportToFileStripMenuItem` нет; в ru.resx остался старый ключ `toolStripMenuItem1` = «Экспорт спектра в файл», он не применяется) — `SaveSpectrumToFile()` через SaveFileDialog — `MainForm.cs:2889` (логика `MainForm.cs:2477`)
- **Экспортировать спектр фона в файл / Export Background Spectrum to File** — `SaveBGSpectrumToFile()`: сохраняет фоновый спектр активного документа в файл — `MainForm.cs:2894` (логика `MainForm.cs:2429`)
- **Жёсткий вычет в файл / Hard subtract to File** — `SaveHardSubtractSpectrumToFile()`: спектр минус фон, результат в файл — `MainForm.cs:2899` (логика `MainForm.cs:2380`)
- **Изменить число каналов / Change channel number** — диалог `ChanNumberChangeDialog` → `ConcatSpectrums()`: пересборка спектра (сжатие/восстановление каналов) в новый документ — `MainForm.cs:2940` (логика `MainForm.cs:1468`)
- **Ограничить энергию или канал / Cutoff energy or channel** — диалог `SpectrumCutOffDialog` → `Cutoff()`: обрезка спектра по энергии или каналу в новый документ — `MainForm.cs:2967` (логика `MainForm.cs:3006`)
- **Нормализовать спектр по ROI / Normalize spectrum with ROI** — диалог `SelectROIDialog` → `NormalizeSpectrum()`; доступен только при привязанной конфигурации устройства — `MainForm.cs:2904` (логика `MainForm.cs:1507`)
- **Применить коррекцию мёртвого времени / Apply dead time correction** — диалог `SelectDeviceDialog` → пересчёт `LiveTime` по мёртвому времени выбранного устройства (`Utils.LiveTime.Calculate`) — `MainForm.cs:2913`
- **Автосохранение спектра / Autosave spectrum** (флажок, CheckOnClick) — переключает `activeDocument.AutoSave`; при включении сразу сохраняет документ — `MainForm.cs:1675`
- ─────
- **Начало измерения / &Start Measurement** — `Ctrl+M` — `dcControlPanel.StartMeasurement()` — `MainForm.cs:3038`
- **Остановить измерение / S&top Measurement** — `Ctrl+P` — `dcControlPanel.StopMeasurement()` — `MainForm.cs:3044`
- **Очистить спектр / &Clear Spectrum** — `dcControlPanel.ClearMeasurementResult()` — `MainForm.cs:3050`

### Вид / &View (`viewTToolStripMenuItem`)
Первые девять пунктов показывают докируемые панели (см. раздел «Панели DockContent»).

- **Управление измерением / Measurement &Control** — показывает `DCControlPanel` — `MainForm.cs:758`
- **Информация об образце / &Sample Information** — показывает `DCSampleInfoView` — `MainForm.cs:802` (логика `MainForm.cs:788`)
- **Список спектров / Spectrum &List** — показывает `DCSpectrumListView` — `MainForm.cs:808`
- **Калибровка энергии / &Energy Calibration** — снимает флаг активной калибровки с прочих документов и показывает `DCEnergyCalibrationView` — `MainForm.cs:832` (обработчик исторически называется `toolStripMenuItem4_Click`)
- **Обнаружение пиков / Peak &Detection** — показывает `DCPeakDetectionView` — `MainForm.cs:822`
- **&Монитор импульсов / &Pulse Monitor** — показывает `DCPulseView` и подключает его к `PulseDetector` активного документа — `MainForm.cs:772`
- **Скорость счёта / Counts R&ate** — показывает `DCCountRateView` + обновляет скорость счёта и статус детектора — `MainForm.cs:885` (обработчик `toolStripMenuItem5_Click`)
- **Калибровка по FWHM / FWHM Calibration** — показывает `DCFwhmCalibrationView` — `MainForm.cs:670` (логика `MainForm.cs:661`)
- **Результат измерений / Measurement &Result** — создаёт новый экземпляр `DCResultView` (не более 4 одновременно) — `MainForm.cs:897`
- ─────
- **Расположение / Layout** — **СКРЫТ В РАНТАЙМЕ**: `MainForm.cs:159-160` безусловно ставит `Visible = false` пункту и разделителю (осознанно, после InitializeComponent); режимы из меню недоступны. (`toolStripMenuItem7`, подменю; галка на текущем режиме — `MainForm.cs:3145`):
  - **Пользовательский режим / User Mode** — сохраняет текущую раскладку в XML, переключает `LayoutMode.UserMode`, пересоздаёт панели (`InitializeToolViews`) и грузит сохранённую раскладку — `MainForm.cs:3173`
  - **Экспертный режим / Expert Mode** — то же для `LayoutMode.ExpertMode` (файл `ExpertMode.xml` в каталоге раскладок) — `MainForm.cs:3201`
- ─────
- **Язык / &Language** (подменю; галка на текущем — `MainForm.cs:3123`; каждый пункт просит перезапуск):
  - **(OS Default)** (RU-ключа нет) — `GlobalConfig.Language = "OS"` — `MainForm.cs:3099`
  - **English** (RU-ключа нет) — `Language = ""` (нейтральная локаль) — `MainForm.cs:3107`
  - **Russian** (RU-ключа нет; имя пункта `jaJPToolStripMenuItem` — наследие японской локали оригинала) — `Language = "ru-RU"` — `MainForm.cs:3115`

### Инструменты / &Tool (`toolsTToolStripMenuItem1`)

- **Конфигурация устройства... / Edit &Device Configurations...** — `ShowDeviceConfigForm(null)`: окно конфигураций устройств — `MainForm.cs:914`
- **Редактировать конфигурации &ROI... / Edit &ROI Configurations...** — `ShowROIConfigForm(null)`: окно конфигураций ROI — `MainForm.cs:1177` (логика `MainForm.cs:1183`)
- **Редактировать библиотеку изотопов... / Edit &Nuclide Definitions...** — `ShowNuclideDefinitionForm()`: окно `NuclideDefinitionForm` — `MainForm.cs:1210` (логика `MainForm.cs:1250`)
- **Редактировать наборы изотопов... / Edit Nuclide Sets...** — модальный `NuclideSetForm` — `MainForm.cs:1215` (логика `MainForm.cs:1265`)
- **Конструктор ROI и наборов нуклидов... / ROI and nuclide set builder...** — `ShowRoiWizardForm()`: докируемая панель `RoiWizard.RoiWizardForm` (один экземпляр, HideOnClose; первый показ — плавающим окном по центру; разрешение детектора берётся из FWHM-калибровки активного спектра) — `MainForm.cs:1271` (логика `MainForm.cs:1288-1384`) ← **добавлено веткой roi-wizard**
- **База изотопов... / Isotope base...** — `ShowNucBaseView()`: отдельное окно `NucBase.NucBase` (не док-панель) — `MainForm.cs:1225` (логика `MainForm.cs:1230`)
- **Открыть папку конфигурации приложения в Windows Explorer... / Open config dir in Explorer...** — `Process.Start("explorer.exe", userDirectoryConfig)` — `MainForm.cs:1220`
- ─────
- **&Настройки / &Preferences** — окно `GlobalConfigForm` (глобальные настройки приложения) — `MainForm.cs:2648`

### Помощь / &Help (`helpHToolStripMenuItem`)

- **&Инструкция / &Manual** — открывает `https://t.me/software_kbradar` в браузере — `MainForm.cs:2865`
- ─────
- **&Проверить обновление... / &Check for updates...** — ClickOnce: `ApplicationDeployment.CheckForDetailedUpdate()`, при наличии — `Update()` + предложение перезапуска — `MainForm.cs:1795`
- **&О BQMoni... / &About...** — модальный `AboutForm` — `MainForm.cs:1787`

### Сводка горячих клавиш главного меню
| Клавиши | Пункт |
|---|---|
| Ctrl+N | Файл → Новый |
| Ctrl+O | Файл → Открыть |
| Ctrl+S | Файл → Сохранить |
| Ctrl+W | Файл → Закрыть |
| Ctrl+M | Спектр → Начало измерения |
| Ctrl+P | Спектр → Остановить измерение |

Других `ShortcutKeys` в `MainForm.Designer.cs` нет.

---

## 2. Тулбары (окно спектра DocEnergySpectrum)

У MainForm тулбара нет. Каждое окно спектра несёт два ToolStrip в нижней панели
(`DocEnergySpectrum.Designer.cs:132,382`). Split-кнопки: клик по кнопке циклически переключает режим,
стрелка раскрывает список. Обработчики — в `DocEnergySpectrum.cs`.

**toolStrip1** (`Ось Y:` … `Ось X:`):
- **Вертикальная единица** (`toolStripSplitButton1`): Имп. / Counts (`DocEnergySpectrum.cs:894`), Имп. в сек. (CPS) (`:886`)
- **Вертикальный масштаб** (`toolStripSplitButton2`): Линейный Y (`:980`), Степень Y / Power Scale (`:971`), Логарифм. Y (`:963`)
- **Поле степени** (`toolStripNumericUpdown`) — степень корня POW; клавиша `e` вводит константу Эйлера (подсказка ресурса)
- **Автомасштаб** (`toolStripSplitButton5`): Без авто масштабирования (`:1082`), Авто масштаб (`:1066`), Авто масштаб по фону (`:1074`)
- **Горизонтальная единица** (`toolStripSplitButton4`): Канал (`:1188`), Энергия (keV) (`:1180`)
- **Увеличение масштаба** (`toolStripButton2`) — `view.ZoominSelectedRegion()` — `:1354`
- **Показать все каналы** (`toolStripButton1`) — `view.FitHorizontalScale()` — `:1337`
- **Масштаб 1:1** (`toolStripButton3`) — `view.SetScale11()` — `:1342`
- **Поле масштаба X** (`toolStripNumUpDownScale`) — изменение масштаба по оси X
- **Калибровка по энергиям** (`toolStripSplitButton10`): Включить (`:1289`) / Выключить (`:1296`) инструмент калибровки (кнопка — переключатель, `:1263`)

**toolStrip2** (`Просмотр:` … `Изменить:`):
- **Режим фона** (`toolStripSplitButtonBgMode`, кнопка циклит `:706`): Показать фоновый спектр (`:774`), Скрыть фоновый спектр (`:784`), Показать спектр за вычетом фона (`:793`), Показать континуум с пиками (`:802`), Показать нормализованный по эффективности спектр (`:811`)
- **Режим отрисовки** (`toolStripSplitButton8`): Сглаживание / Antialias (`:849`), Обычное отображение (`:857`)
- **Тип графика** (`toolStripSplitButton3`): Столбцы (`:1026`), Линии (`:1018`)
- **Сглаживание** (`toolStripSplitButton6`): Нет (`:1124`), Скользящее среднее (`:1133`), Скользящее среднее с весом (`:1142`)
- **Обнаруженные пики** (`toolStripSplitButton9`): Показать (`:1241`, включает детекцию пиков), Скрыть (`:1252`, выключает)
- **Снять скрин спектра в файл** (`toolStripScreenShotButton`) — `view.takeScreenshot()` — `:1383`
- **Сохранить текущий документ** (`toolStripSaveButton`) — `:1416`
- **Перезагрузить фон из файла** (`toolStripRefreshBgButton`) — при пустом пути к фону сначала диалог выбора — `:1388`
- **Отделить фон от спектра** (`toolStripSplitButton`) — `DocumentManager.SplitDocEnergySpectrum()`: документ разбивается на список «спектр + фон» — `:1428`

---

### 2.1 Статус-строка мастера ROI (StatusStrip с кнопками)

`RoiWizard\RoiWizardForm.Designer.cs:184-188, 980-991`: три `ToolStripButton` — Справка (`buttonHelp`, справа),
«◂ Назад» (`buttonStepPrev`), «Вперёд ▸» (`buttonStepNext`); RU-тексты из `RoiWizardStrings`, обработчики
`RoiWizardForm.cs:645-647` (`ShowHelp()`, `GoStep(±1)`), enable/текст-логика :2413-2420. Уточнение к тезису
«тулбары живут только в окне спектра»: ToolStrip-кнопки есть и здесь.

## 3. Контекстные меню

Всего в проекте пять `ContextMenuStrip` (по grep): `DocEnergySpectrum`, `DCPeakDetectionView`, `ToolWindow`,
`Utils\CalibrationGraph`, `Utils\FWHMCalibrationGraph`. Шестое семейство — библиотечный `HeaderContextMenu` XPTable
(наследник старого ContextMenu, потому grep по ContextMenuStrip его не видит): правый клик по заголовку колонки
любой таблицы приложения — переключение видимости колонок + «More...» (`XPTable\Models\Table.cs:832, 7176-7178`;
включён по умолчанию). В клиентской области списка спектров (`DCSpectrumListView`) и таблицы результатов
(`DCResultView`) контекстных меню нет — там кнопки (Новый/Удалить/Импорт/Экспорт и «Ред. ROI...»).

### 3.1 Спектр — правый клик по окну DocEnergySpectrum (`contextMenuStrip1`, `DocEnergySpectrum.Designer.cs:599`)
- **Увеличить масштаб (&M) / Increase selection (&M)** — `view.ZoominSelectedRegion()` — `DocEnergySpectrum.cs:1360`
- **Показать все каналы (&A) / Show all channels (&A)** — `view.FitHorizontalScale()` — `DocEnergySpectrum.cs:1348`
- ─────
- **Конфигурация устройства (&D) / Device configuration (&D)** (подменю):
  - **Установить нижний порог спектра по курсору (&L)** — событие `SetLowerThreshold` → MainForm открывает `DeviceConfigForm` и пишет порог из позиции курсора в конфигурацию устройства — `DocEnergySpectrum.cs:1375` → `MainForm.cs:1993`
  - **Установить верхний порог спектра по курсору (&H)** — аналогично для верхнего порога — `DocEnergySpectrum.cs:1452` → `MainForm.cs:2005`
- **Конфигурация ROI (&R) / ROI Configuration (&R)** (подменю):
  - **Создать ROI из выбранного диапазона (&S)** — событие `CreateNewROI` → MainForm открывает `ROIConfigForm` и создаёт определение ROI из границ выделения — `DocEnergySpectrum.cs:1366` → `MainForm.cs:1967`
- ─────
- **Сохранить (&S) / Save (&S)** — событие `SaveDocument` — `DocEnergySpectrum.cs:517`
- **Закрыть(&C) / Close (&C)** (в ресурсе без пробела) — событие `CloseDocument` — `DocEnergySpectrum.cs:523`

### 3.2 Таблица обнаруженных пиков — DCPeakDetectionView (`contextMenuStrip1`, `DCPeakDetectionView.Designer.cs:167`)
Оба пункта активны только при выделенных строках (`DCPeakDetectionView.cs:486`).
- **Добавить в калибровку / Add in calibration** — для каждой выбранной строки берёт канал и энергию (минус ошибка) и зовёт `mainForm.addCalibration()` — `DCPeakDetectionView.cs:447`
- **Поиск в Базе Изотопов / Search Isotope Base** — `mainForm.CallNucBaseSearch(энергия строки)`: открывает Базу изотопов с поиском по энергии — `DCPeakDetectionView.cs:480`

### 3.3 График калибровки энергии — CalibrationGraph (отдельное модальное окно, открывается кнопкой из DCEnergyCalibrationView — `DCEnergyCalibrationView.cs:694-699`, ShowDialog; `Utils\CalibrationGraph.Designer.cs:43`)
- **Сохранить новые точки калибровки / Save new calibration points** — записывает отредактированные точки в `ActiveResultData.CalibrationPoints` — `Utils\CalibrationGraph.cs:360`
- **Отменить изменения / Reset modifications** — откат точек и полинома к исходным — `Utils\CalibrationGraph.cs:367`

### 3.4 График FWHM-калибровки — FWHMCalibrationGraph (отдельное модальное окно, кнопка из DCFwhmCalibrationView — `DCFwhmCalibrationView.cs:906-910`, ShowDialog; `Utils\FWHMCalibrationGraph.Designer.cs`). RU-ключей нет — файла `FWHMCalibrationGraph.ru.resx` не существует, в русской локали пункты по-английски:
- **Save new calibration points (Сохранить новые точки)** — пишет пики в `ActiveResultData.FwhmCalibration.CalibrationPeaks` — `Utils\FWHMCalibrationGraph.cs:381`
- **Reset modifications (Отменить изменения)** — откат — `Utils\FWHMCalibrationGraph.cs:389`

### 3.5 ToolWindow (базовый класс панелей) — мёртвое меню
`ToolWindow.Designer.cs:30` создаёт `contextMenuStrip1` с пунктами **Option&1 / Option&2 / Option&3**,
но меню никому не присвоено (`ContextMenuStrip`/`TabPageContextMenuStrip` не назначаются) и обработчиков Click нет —
наследие примера DockPanel Suite, в интерфейсе не появляется.

---

## 4. Панели DockContent

Все инструментальные панели — наследники `ToolWindow : DockContent` (WeifenLuo DockPanel Suite);
создаются при старте в `MainForm.InitializeToolViews()` — `MainForm.cs:252` — и восстанавливаются из XML-раскладки
(`ExpertMode.xml`, режимы «Пользовательский/Экспертный»). Повторное открытие — из меню **Вид**.

| Панель | Заголовок RU / EN | Как открывается | Примечание |
|---|---|---|---|
| `DCControlPanel` | Управление измерением / Measurement Control | Вид → Управление измерением (`MainForm.cs:758`) | запуск/остановка/очистка измерения, preset time, выбор устройства и ROI |
| `DCSampleInfoView` | Информация об образце / Sample Information | Вид → Информация об образце (`:802`) | |
| `DCSpectrumListView` | Список спектров / Spectrum List | Вид → Список спектров (`:808`) | кнопки Новый/Удалить/Импорт/Экспорт |
| `DCEnergyCalibrationView` | Калибр. Энергии / Energy Calibration | Вид → Калибровка энергии (`:832`) | содержит график `CalibrationGraph` (контекстное меню 3.3) |
| `DCFwhmCalibrationView` | FWHM Калибровка / FWHM Calibration | Вид → Калибровка по FWHM (`:670`) | содержит `FWHMCalibrationGraph` (меню 3.4) |
| `DCPeakDetectionView` | Обнаружение пиков / Peak Detection | Вид → Обнаружение пиков (`:822`) | таблица пиков с контекстным меню 3.2; выбор набора нуклидов, деконволюция |
| `DCPulseView` | Форма импульса / Pulse Shape | Вид → Монитор импульсов (`:772`) | |
| `DCCountRateView` | Скорость счёта / Counts Rate | Вид → Скорость счёта (`:885`) | |
| `DCResultView` | Результат измерений / Measurement Result | Вид → Результат измерений (`:897`) | до 4 экземпляров; кнопка «Ред. ROI...» |
| `DocEnergySpectrum` | имя файла спектра | Файл → Новый/Открыть, импорт, drag&drop | документ (DockAreas.Document), контекстное меню 3.1 и два тулбара |
| `RoiWizard.RoiWizardForm` | Конструктор ROI и наборов нуклидов / ROI and nuclide set builder | Инструменты → Конструктор ROI... (`:1271`) | **roi-wizard**; DockContent с HideOnClose, один экземпляр, вкладки «1 · Изотопы», «2 · Линии», «3 · Оформление и создание»; восстанавливается из раскладки через `GetContentFromPersistString` |

Не DockContent (обычные окна): `NucBase.NucBase` (База изотопов), `ROIConfigForm`, `NuclideDefinitionForm`,
`NuclideSetForm`, `DeviceConfigForm`, `GlobalConfigForm`, `AboutForm`, `EffCalcMCDialog`, диалоги выбора.

---

## 5. Пункты ветки roi-wizard

1. **Инструменты → Конструктор ROI и наборов нуклидов...** (`RoiWizardToolStripMenuItem`, `MainForm.Designer.cs:593-597`) — обработчик `MainForm.cs:1271`, показ панели `MainForm.cs:1312`, ленивая инициализация с проверкой доступности базы нуклидов `MainForm.cs:1288`, расчёт разрешения детектора `MainForm.cs:1351`.
2. **Файл → Импорт → LSRM EffCalcMC.txt -> ROI** (`EffCalcMCFileToolStripMenuItem`, `MainForm.Designer.cs:225-229`) — обработчик `MainForm.cs:2529-2554`: импорт кривой эффективности EffCalcMC в конфигурацию ROI.

---

## 6. Не выяснено / тёмные места

- **Осиротевшие RU-ключи в `MainForm.ru.resx`**: `toolStripMenuItem1` («&Экспорт спектра в файл»), `toolStripMenuItem2` («Отчёт...»), `toolStripMenuItem3` («Статус измерений»), `toolStripMenuItem4` («&Калибровка энергии»), `toolStripMenuItem5` («Скорость С&чёта») — пунктов с такими именами в `MainForm.Designer.cs` нет (пункты были переименованы или удалены; «Отчёт...» и «Статус измерений» в текущем меню отсутствуют вовсе). Следствие: пункт «Export Spectrum to File» в русской локали отображается по-английски.
- **Пункты без русских подписей** (fallback на EN в русской локали): LSRM EffCalcMC.txt -> ROI, SpectraLine GBS, (OS Default), English, Russian, Export Spectrum to File.
- **`jaJPToolStripMenuItem`** — имя от японской локали оригинального BecquerelMonitor; фактический текст «Russian», ставит `ru-RU`.
- Осиротевший ключ `toolStripSplitButton7.ToolTipText` = «Выбрать отображение фона» (`DocEnergySpectrum.ru.resx:294`) — кнопка переименована в `toolStripSplitButtonBgMode`.
- Мёртвый обработчик `ToolStripStatusLabel2_Click` (`MainForm.cs:2213-2222`, опрос температуры AtomSpectra кликом по метке статус-строки) — нигде не подписан, недостижим.
- Оба режима раскладки делят один файл: `LayoutConfigFile()` всегда возвращает `ExpertMode.xml` (`MainForm.cs:3229-3232`).
- **Файл → Экспорт → для FWHM** — числа 530/627/780 кэВ и кодировка вывода 932 (Shift-JIS) зашиты в код (`MainForm.cs:3056-3085`); назначение — экспорт окрестности пика 662 кэВ для внешней оценки FWHM, но в меню это никак не пояснено.
- **ToolWindow.contextMenuStrip1 (Option1/2/3)** — заготовка без обработчиков, нигде не привязана.
- Вкладка мастера в этой ветке называется «2 · Линии» (`RoiWizard\RoiWizardStrings.ru.resx`); переименование «2 · Цепочки» существует только в веб-версии конструктора, в WinForms-порт не переносилось.
