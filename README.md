# Проект рефакторинга BecqMoni

Рабочий репозиторий проекта пересмотра архитектуры [BecqMoni](https://github.com/Am6er/BecqMoni).

**Сайт:** https://vibeengineering-llc.github.io/becqmoni-arch-notes/

База обсуждения — arch-документ автора BecqMoni ([arch/architecture-refactor.md](https://github.com/Am6er/BecqMoni/blob/roi-wizard-reworked/arch/architecture-refactor.md), PR #32). Здесь — техническое задание, схемы и аналитические материалы, на которых оно построено.

## Состав

| Страница | Содержание |
|---|---|
| [ТЗ рефакторинга](https://vibeengineering-llc.github.io/becqmoni-arch-notes/bm-arch-tz.html) | **Центральный документ**: концепт (живой расчёт + воспроизводимость результата), схема тракта анализа, три уровня активности, Волна 0 и девять этапов в двух частях, правила совместимости, 14 сценариев приёмки |
| [Волна 0: дефекты](https://vibeengineering-llc.github.io/becqmoni-arch-notes/wave0.html) | Реестр из 15 дефектов, дающих неверные числа сегодня, без смены модели данных; атрибуция находки при каждом пункте |
| [Лучшие решения](https://vibeengineering-llc.github.io/becqmoni-arch-notes/best-solutions.html) | Функциональная карта экосистем BecqMoni и LSRM, 22 наиболее удачных решения с оценкой переносимости |
| [BecqMoni ↔ SpectraLine](https://vibeengineering-llc.github.io/becqmoni-arch-notes/comparison.html) | Сравнительный анализ архитектуры, функционала и алгоритмов (каждая ячейка BM-колонки — с адресом файл:строка) |
| [Карта связей BecqMoni](https://vibeengineering-llc.github.io/becqmoni-arch-notes/arch-map.html) | Реестр сущностей, матрица связей, неявные конвенции, потоки данных — по коду ветки roi-wizard-reworked |
| [Меню BecqMoni](https://vibeengineering-llc.github.io/becqmoni-arch-notes/menu-map.html) | Полное дерево меню с обработчиками, контекстные меню, тулбары, док-панели |
| [SpectraLineBG](https://vibeengineering-llc.github.io/becqmoni-arch-notes/spectraline.html), [NuclideMaster](https://vibeengineering-llc.github.io/becqmoni-arch-notes/nuclidemaster.html) | Конспекты документации референсного пакета ЛСРМ |

Каждая страница скачивается в Markdown (ссылка в шапке). Схемы — в `figures/`.

## Провенанс

Материалы о BecqMoni построены по открытому коду проекта Am6er/BecqMoni (вершина d80c7ee, ветка roi-wizard-reworked); фактические утверждения прошли перекрёстный аудит с независимой сверкой адресов. Конспекты SpectraLine/NuclideMaster — сжатый пересказ пользовательской документации, которую правообладатель раздаёт открыто: [SpectraLine](https://lsrm.ru/help/dokumentacija/spectraline.php), [общие модули](https://lsrm.ru/help/dokumentacija/obshchie_moduli.php) (в т.ч. «Алгоритмические основы» и Efficiency), [Nuclide Master](https://lsrm.ru/help/dokumentacija/nuclidemaster.php). Правообладатель документации — ООО «ЛСРМ» (lsrm.ru); ссылки на страницы PDF в конспектах указывают на эти оригиналы.
