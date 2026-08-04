Предлагаемый план реализации

В FlatTable:
Добавляем опциональные пропсы infinite, hasMore, isLoadingMore, onLoadMore
Если infinite === true — рендерим внизу таблицы sentinel + маленький индикатор загрузки
Подключаем IntersectionObserver
При появлении sentinel и hasMore && !isLoadingMore вызываем onLoadMore()

В CatalogeSborn:
Убираем <Pagination>
Данные начинаем накапливать (setFilteredSbornData(prev => [...prev, ...newResults]))
При смене фильтра / клике по дереву — полностью сбрасываем данные и страницу
Передаём в FlatTable новые пропсы

Защита от лишних запросов:
Флаг isLoadingMore
Проверка hasMore
Сброс при смене newident / фильтров



Вопрос перед стартом
Подтверждаешь, что идём по Варианту B (добавляем опциональный infinite в FlatTable)?
Если да — сразу приступлю к реализации (сначала покажу изменения в FlatTable, потом в CatalogeSborn).идём по Варианту B. + Подробные комментарии и объяснения работы кодаОтлично. Идём по Варианту B.
Ниже — полная реализация с подробными комментариями.

1. Обновлённый FlatTable.jsx
Добавлены только опциональные пропсы. Все существующие места использования продолжают работать без изменений.
jsximport React, { useEffect, useMemo, useState, useRef, useCallback } from 'react';
import {
  useReactTable,
  getCoreRowModel,
  flexRender,
  getSortedRowModel
} from "@tanstack/react-table";

import AddCircleOutlineIcon from "@mui/icons-material/AddCircleOutline";
import EditIcon from "@mui/icons-material/Edit";
import DeleteForeverIcon from '@mui/icons-material/DeleteForever';
import FilterListIcon from "@mui/icons-material/FilterList";
import RefreshIcon from "@mui/icons-material/Refresh";
import { Box, CircularProgress, Typography } from '@mui/material';

import FilterDialog from '../../../pages/FilterDialog';
import { formatDate } from '../../../utils/dateUtils';

/**
 * Универсальная плоская таблица с поддержкой:
 * - оптимистичного обновления
 * - фильтров
 * - CRUD-формы
 * - опционального бесконечного скролла (infinite scroll)
 *
 * -------------------------------------------------
 * Основные пропсы (как раньше):
 * @param {Array} data
 * @param {Array} columns
 * @param {String} pk
 * @param {Function} setData
 * @param {Function} reloadData
 * @param {React.Component} FormComponent
 * @param {String} initialNewIdent
 * @param {Function} onApplyFilters
 * @param {boolean} loading
 * @param {Function} onRowSelect
 *
 * Новые опциональные пропсы для бесконечной подгрузки:
 * @param {boolean} infinite        - включить режим infinite scroll (по умолчанию false)
 * @param {boolean} hasMore         - есть ли ещё данные на сервере
 * @param {boolean} isLoadingMore   - идёт ли сейчас подгрузка следующей порции
 * @param {Function} onLoadMore     - колбэк, который вызывается, когда нужно загрузить следующую страницу
 */
const FlatTable = ({
  title = "",
  visibleToolbarButtons = ['add', 'filter', 'refresh'],
  data,
  columns,
  pk,
  setData,
  reloadData,
  FormComponent,
  initialNewIdent = "",
  onApplyFilters,
  loading = false,
  FilterComponent,
  filterConfig = {},
  onRowSelect,

  // ===== Новые пропсы для infinite scroll =====
  infinite = false,          // по умолчанию выключен → старое поведение
  hasMore = false,
  isLoadingMore = false,
  onLoadMore = () => {},
}) => {

  // ============================================================
  // 1. Форматирование колонок (даты)
  // ============================================================
  // Делаем один раз и переиспользуем. Раньше логика была продублирована.
  const formattedColumns = useMemo(() => {
    return columns.map(column => {
      if (column.type === 'date') {
        return {
          ...column,
          // Если у колонки уже есть свой cell — оставляем его,
          // иначе добавляем форматирование даты.
          cell: column.cell
            ? column.cell
            : (props) => formatDate(props.getValue()),
        };
      }
      return column;
    });
  }, [columns]);

  // ============================================================
  // 2. Состояния компонента
  // ============================================================
  const [selectedRecord, setSelectedRecord] = useState(null);
  const [rowSelection, setRowSelection] = useState({});
  const [sorting, setSorting] = useState([]);
  const [mode, setMode] = useState(null);
  const [isFormOpen, setIsFormOpen] = useState(false);
  const [isFilterDialogOpen, setIsFilterDialogOpen] = useState(false);

  const [filterParams, setFilterParams] = useState({
    newident: initialNewIdent
  });

  const [isSimpleFilterActive, setIsSimpleFilterActive] = useState(false);

  // ----- Infinite scroll -----
  // ref на невидимый элемент в самом низу таблицы.
  // Когда он появляется в зоне видимости — грузим следующую порцию.
  const sentinelRef = useRef(null);

  // ============================================================
  // 3. Синхронизация initialNewIdent
  // ============================================================
  useEffect(() => {
    setFilterParams(prev => ({ ...prev, newident: initialNewIdent }));
  }, [initialNewIdent]);

  // ============================================================
  // 4. Intersection Observer (сердце infinite scroll)
  // ============================================================
  /**
   * Как это работает:
   * 1. Создаём observer, который следит за sentinel-элементом.
   * 2. Когда sentinel пересекает область видимости (с запасом rootMargin),
   *    и при этом hasMore === true и загрузка сейчас не идёт —
   *    вызываем onLoadMore().
   * 3. Observer автоматически отключается/включается при изменении зависимостей.
   *
   * root: null → используем viewport браузера.
   * rootMargin: '200px' → начинаем грузить чуть заранее (за 200px до конца).
   */
  useEffect(() => {
    // Если infinite выключен — ничего не делаем
    if (!infinite) return;

    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    const observer = new IntersectionObserver(
      (entries) => {
        const [entry] = entries;

        // entry.isIntersecting === true → элемент виден
        if (entry.isIntersecting && hasMore && !isLoadingMore) {
          onLoadMore();
        }
      },
      {
        root: null,           // viewport
        rootMargin: '200px',  // подгружаем заранее
        threshold: 0.1
      }
    );

    observer.observe(sentinel);

    // Очистка при размонтировании или изменении зависимостей
    return () => {
      if (sentinel) {
        observer.unobserve(sentinel);
      }
    };
  }, [infinite, hasMore, isLoadingMore, onLoadMore]);

  // ============================================================
  // 5. Обработчики
  // ============================================================
  const handleRowClick = (row) => {
    setSelectedRecord(row.original);
    if (onRowSelect) {
      onRowSelect(row.original);
    }
  };

  const handleAddClick = () => {
    setSelectedRecord({ initialNewIdent });
    setMode("add");
    setIsFormOpen(true);
  };

  const handleEditClick = (record) => {
    setSelectedRecord(record);
    setMode("edit");
    setIsFormOpen(true);
  };

  const handleDeleteClick = (record) => {
    setSelectedRecord(record);
    setMode("delete");
    setIsFormOpen(true);
  };

  const handleOpenFilterDialog = () => {
    setIsFilterDialogOpen(true);
  };

  const handleApplyFilters = (filters, filterType) => {
    const newFilterParams = { ...filters };

    setFilterParams(newFilterParams);
    setIsFilterDialogOpen(false);

    if (filterType === 'simple') {
      setIsSimpleFilterActive(true);
    } else {
      setIsSimpleFilterActive(false);
    }

    if (onApplyFilters) {
      onApplyFilters(newFilterParams);
    } else {
      reloadData(newFilterParams);
    }
  };

  const handleReload = () => {
    if (isSimpleFilterActive) {
      setFilterParams({});
      setIsSimpleFilterActive(false);
      reloadData();
    } else {
      reloadData(filterParams);
    }
  };

  /**
   * Успешное сохранение (add / edit / delete).
   * Работает с локальным массивом data — оптимистичное обновление.
   */
  const handleSuccess = (record, mode) => {
    if (mode === "add") {
      setData(prev => [...prev, record]);
      setSelectedRecord(record);
      setRowSelection({ [record[pk]]: true });
    } else if (mode === "edit") {
      setData(prev => prev.map(item => item[pk] === record[pk] ? record : item));
      setSelectedRecord(record);
      setRowSelection({ [record[pk]]: true });
    } else if (mode === "delete") {
      setData(prev => prev.filter(item => item[pk] !== record[pk]));
      setSelectedRecord(null);
      setRowSelection({});
    }
    setIsFormOpen(false);
  };

  const handleFormClose = () => {
    setIsFormOpen(false);
  };

  // ============================================================
  // 6. Колонки таблицы (с колонкой действий)
  // ============================================================
  const tableColumns = useMemo(() => {
    return [
      ...formattedColumns,
      {
        id: "actions",
        header: "Действия",
        cell: ({ row }) => (
          <div style={{ display: "flex", gap: "10px" }}>
            <EditIcon
              titleAccess="Редактирование"
              sx={{ width: 20, cursor: "pointer" }}
              onClick={() => handleEditClick(row.original)}
            />
            <DeleteForeverIcon
              titleAccess="Удаление"
              sx={{ width: 20, cursor: "pointer" }}
              onClick={() => handleDeleteClick(row.original)}
            />
          </div>
        )
      }
    ];
  }, [formattedColumns]);

  // ============================================================
  // 7. Конфигурация TanStack Table
  // ============================================================
  const table = useReactTable({
    data,
    columns: tableColumns,
    state: { sorting, rowSelection },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  // ============================================================
  // 8. Рендер
  // ============================================================
  return (
    <div style={{ border: "1px solid #ccc" }}>
      {/* Панель инструментов */}
      <div
        className='tools-panel'
        style={{
          margin: '4px',
          display: "flex",
          position: "sticky",
          top: 0,
          zIndex: 1,
          background: "#fff"
        }}
      >
        {visibleToolbarButtons.includes('add') && (
          <AddCircleOutlineIcon
            titleAccess="Добавить"
            sx={{ width: 30, cursor: 'pointer' }}
            onClick={handleAddClick}
          />
        )}
        {visibleToolbarButtons.includes('filter') && (
          <FilterListIcon
            titleAccess="Фильтр"
            sx={{ width: 30, cursor: 'pointer', marginLeft: "10px" }}
            onClick={handleOpenFilterDialog}
          />
        )}
        {visibleToolbarButtons.includes('refresh') && (
          <RefreshIcon
            titleAccess="Обновить"
            style={{ width: 30, cursor: 'pointer', marginLeft: "10px" }}
            onClick={handleReload}
          />
        )}
        <Typography className='flex-1 text-center font-bold text-lg'>
          {title}
        </Typography>
      </div>

      {/* Состояние загрузки (первая порция) */}
      {loading ? (
        <Box sx={{ display: "flex", justifyContent: "center", p: 4 }}>
          <CircularProgress />
        </Box>
      ) : (
        <>
          <table
            border="1"
            cellPadding="5"
            style={{ width: '100%', borderCollapse: 'collapse' }}
          >
            <thead>
              {table.getHeaderGroups().map((headerGroup) => (
                <tr key={headerGroup.id}>
                  {headerGroup.headers.map((header) => (
                    <th
                      key={header.id}
                      style={{
                        position: "sticky",
                        top: 0,
                        zIndex: 1,
                        background: "#f0f0f0",
                        border: "1px solid #ccc",
                        padding: "5px",
                        cursor: 'pointer'
                      }}
                      onClick={header.column.getToggleSortingHandler()}
                    >
                      {header.isPlaceholder
                        ? null
                        : flexRender(header.column.columnDef.header, header.getContext())
                      }
                      {{ asc: '↑', desc: '↓' }[header.column.getIsSorted()] ?? null}
                    </th>
                  ))}
                </tr>
              ))}
            </thead>

            <tbody>
              {table.getRowModel().rows.map((row) => (
                <tr
                  key={row.id}
                  onClick={() => handleRowClick(row)}
                  style={{
                    cursor: 'pointer',
                    backgroundColor:
                      selectedRecord?.[pk] === row.original[pk]
                        ? "#e6f7ff"
                        : "transparent",
                  }}
                >
                  {row.getVisibleCells().map((cell) => (
                    <td key={cell.id}>
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>

          {/* ============================================================
              INFINITE SCROLL: sentinel + индикатор
              ============================================================
              Этот блок рендерится только когда infinite={true}.

              1. sentinel — невидимый div в самом низу.
                 IntersectionObserver следит именно за ним.
              2. Когда isLoadingMore === true — показываем спиннер.
              3. Когда данные закончились (hasMore === false) —
                 можно показать текст "Больше записей нет".
          */}
          {infinite && (
            <div style={{ padding: '12px 0', textAlign: 'center' }}>
              {/* Невидимый сентинел */}
              <div
                ref={sentinelRef}
                style={{ height: '1px', width: '100%' }}
              />

              {/* Индикатор подгрузки следующей порции */}
              {isLoadingMore && (
                <Box sx={{ display: 'flex', justifyContent: 'center', py: 1 }}>
                  <CircularProgress size={24} />
                </Box>
              )}

              {/* Сообщение, что всё загружено */}
              {!hasMore && data.length > 0 && (
                <Typography variant="body2" color="text.secondary">
                  Все записи загружены
                </Typography>
              )}
            </div>
          )}
        </>
      )}

      {/* Диалог фильтрации */}
      {FilterComponent ? (
        <FilterComponent
          open={isFilterDialogOpen}
          onClose={() => setIsFilterDialogOpen(false)}
          onApply={handleApplyFilters}
          {...filterConfig}
        />
      ) : (
        <FilterDialog
          open={isFilterDialogOpen}
          onClose={() => setIsFilterDialogOpen(false)}
          onApply={handleApplyFilters}
          izdatOptions={[]}
          initialFilters={filterParams}
        />
      )}

      {/* CRUD-форма */}
      {isFormOpen && FormComponent && (
        <FormComponent
          record={selectedRecord}
          mode={mode}
          onClose={handleFormClose}
          onSuccess={handleSuccess}
          open={isFormOpen}
        />
      )}
    </div>
  );
};

export default FlatTable;

2. Обновлённый CatalogeSborn.jsx
Основные изменения:

Убрана пагинация MUI
Данные накапливаются
При смене фильтра / клике по дереву — полный сброс
Передаются новые пропсы в FlatTable

jsximport React, { useEffect, useState, useMemo, useCallback } from "react";
import HierarchicalTable from "../comComponents/UI/table/HierarchicalTable";
import FlatTable from "../components/UI/table/FlatTable";
import EditDeleteFormSborn from "../EditDeleteFormSborn";
import { api } from "../config/api";
import {
  Panel,
  PanelGroup,
  PanelResizeHandle,
} from "react-resizable-panels";

// ==================================================
// CatalogeSborn — каталог сборников с бесконечной подгрузкой
// ==================================================
export default function CatalogeSborn() {
  // ===============================================
  // STATE
  // ===============================================
  const [dataCat, setDataCat] = useState([]);
  const [activeHierId, setActiveHierId] = useState(null);

  const [filterIndex, setFilterIndex] = useState(null);
  const [filterParams, setFilterParams] = useState({});

  // Данные таблицы (теперь накапливаются)
  const [filteredSbornData, setFilteredSbornData] = useState([]);

  const [loading, setLoading] = useState(true);          // первая загрузка каталога
  const [loadingSborn, setLoadingSborn] = useState(false); // первая порция сборников
  const [isLoadingMore, setIsLoadingMore] = useState(false); // подгрузка следующих страниц

  const [pageIndex, setPageIndex] = useState(1);
  const [totalCount, setTotalCount] = useState(0);
  const pageSize = 17;

  // Есть ли ещё данные для подгрузки
  const hasMore = filteredSbornData.length < totalCount;

  // ===============================================
  // Загрузка дерева каталога (один раз)
  // ===============================================
  useEffect(() => {
    const fetchCatalog = async () => {
      try {
        setLoading(true);
        const response = await api.get("api/catalog/tree/");
        setDataCat(response.data);
      } catch (error) {
        console.error("Ошибка загрузки каталога:", error);
      } finally {
        setLoading(false);
      }
    };
    fetchCatalog();
  }, []);

  // ===============================================
  // Загрузка сборников
  // ===============================================
  /**
   * Главная функция загрузки.
   *
   * @param {object} params        - дополнительные параметры фильтра
   * @param {number|null} pageOverride - какая страница нужна (если null — берём текущую)
   * @param {boolean} append       - true = добавить к существующим данным,
   *                                 false = полностью заменить (при смене фильтра)
   */
  const loadSbornData = useCallback(
    async (params = {}, pageOverride = null, append = false) => {
      try {
        // Разные флаги загрузки для первой порции и для "ещё"
        if (append) {
          setIsLoadingMore(true);
        } else {
          setLoadingSborn(true);
        }

        const page = pageOverride ?? pageIndex;

        const requestParams = {
          ...filterParams,
          ...params,
          newident: params.newident ?? filterIndex,
          paginate: true,
          page,
          page_size: pageSize,
        };

        // Если пришли новые фильтры — всегда начинаем с 1 страницы
        if (Object.keys(params).length > 0 && !append) {
          requestParams.page = 1;
          setPageIndex(1);
        }

        const query = new URLSearchParams();
        Object.entries(requestParams).forEach(([key, value]) => {
          if (value !== undefined && value !== "" && value !== null) {
            query.append(key, value);
          }
        });

        const response = await api.get(`api/sborn/?${query.toString()}`);
        const results = response.data.results || [];
        const count = response.data.count || 0;

        setTotalCount(count);

        if (append) {
          // Бесконечная подгрузка — добавляем к уже имеющимся данным
          setFilteredSbornData(prev => [...prev, ...results]);
        } else {
          // Первая загрузка или смена фильтра — полностью заменяем
          setFilteredSbornData(results);
        }

        // Синхронизируем номер страницы с ответом сервера (если он есть)
        if (response.data.current_page) {
          setPageIndex(response.data.current_page);
        } else {
          setPageIndex(page);
        }
      } catch (error) {
        console.error("Ошибка загрузки сборников:", error);
      } finally {
        setLoadingSborn(false);
        setIsLoadingMore(false);
      }
    },
    [filterParams, filterIndex, pageIndex, pageSize]
  );

  // ===============================================
  // Клик по дереву
  // ===============================================
  const handleHierRowSelect = (row) => {
    const index = row.index;
    setActiveHierId(row.id);
    setFilterIndex(index);
    setFilterParams({ newident: index });

    // Полный сброс + загрузка первой страницы
    setPageIndex(1);
    loadSbornData({ newident: index }, 1, false);
  };

  // ===============================================
  // Применение фильтров из диалога
  // ===============================================
  const handleApplyFilters = (filters) => {
    setFilterParams(filters);
    setPageIndex(1);
    // Полная замена данных
    loadSbornData(filters, 1, false);
  };

  // ===============================================
  // Подгрузка следующей страницы (вызывается из FlatTable)
  // ===============================================
  const handleLoadMore = useCallback(() => {
    if (!hasMore || isLoadingMore || loadingSborn) return;

    const nextPage = pageIndex + 1;
    loadSbornData({}, nextPage, true); // append = true
  }, [hasMore, isLoadingMore, loadingSborn, pageIndex, loadSbornData]);

  // ===============================================
  // Колонки
  // ===============================================
  const colCat = useMemo(() => [
    { header: "", accessorKey: "level_name", cell: (info) => info.getValue() },
    { header: "№", accessorKey: "element_num", cell: (info) => info.getValue() },
    { header: "Наименование", accessorKey: "element_name", cell: (info) => info.getValue() },
  ], []);

  const colSb = useMemo(() => [
    { accessorKey: "sborn_id", header: "Код сбора", cell: (info) => info.getValue() },
    { accessorKey: "newident", header: "Index", cell: (info) => info.getValue() },
    { accessorKey: "ident", header: "Пред. индекс", cell: (info) => info.getValue() },
    { accessorKey: "name", header: "Наименование", cell: (info) => info.getValue() },
  ], []);

  // ===============================================
  // Рендер
  // ===============================================
  if (loading && dataCat.length === 0) {
    return (
      <div className="flex items-center justify-center h-[50vh]">
        <p>Загрузка каталога...</p>
      </div>
    );
  }

  return (
    <div className="flex flex-col h-screen w-full p-2 overflow-hidden bg-white">
      <h3 className="text-[13px] font-semibold text-center mb-1.5 uppercase tracking-wide">
        Каталог сборников
      </h3>

      <PanelGroup
        direction="horizontal"
        autoSaveId="catsborn-layout"
        className="flex-grow border border-gray-200 rounded-lg"
      >
        {/* Левая панель — дерево */}
        <Panel defaultSize={50} minSize={15}>
          <div className="h-full p-1 overflow-auto">
            <HierarchicalTable
              data={dataCat}
              columns={colCat}
              onRowSelect={handleHierRowSelect}
              expandName="level_name"
              onSelect={setActiveHierId}
              activeId={activeHierId}
            />
          </div>
        </Panel>

        <PanelResizeHandle className="w-[3px] bg-gray-300 hover:bg-blue-400 cursor-col-resize" />

        {/* Правая панель — таблица с бесконечным скроллом */}
        <Panel defaultSize={50} minSize={15}>
          <div className="h-full p-1 bg-gray-50 overflow-auto">
            <FlatTable
              data={filteredSbornData}
              columns={colSb}
              pk="sborn_id"
              setData={setFilteredSbornData}
              reloadData={loadSbornData}
              FormComponent={EditDeleteFormSborn}
              initialNewIdent={filterIndex}
              onApplyFilters={handleApplyFilters}
              loading={loadingSborn}

              // ===== Infinite scroll =====
              infinite={true}
              hasMore={hasMore}
              isLoadingMore={isLoadingMore}
              onLoadMore={handleLoadMore}
            />
          </div>
        </Panel>
      </PanelGroup>
    </div>
  );
}

Как теперь работает поток данных

Пользователь кликает по узлу дерева → handleHierRowSelect
→ сбрасываем страницу в 1 → loadSbornData(..., append=false) → данные полностью заменяются.
Пользователь применяет фильтр → то же самое (полная замена).
Пользователь скроллит вниз правой панели → sentinel становится видимым →
IntersectionObserver срабатывает → handleLoadMore →
loadSbornData(..., page=next, append=true) → новые строки добавляются в конец.
Когда filteredSbornData.length >= totalCount → hasMore = false → подгрузка прекращается, показывается текст «Все записи загружены».


Важные моменты

Существующие места, где используется FlatTable без новых пропсов, продолжают работать как раньше.
При 10 000 записей обычный <table> начнёт подтормаживать. Если заметите лаги — следующим шагом можно добавить виртуализацию (@tanstack/react-virtual).
rootMargin: '200px' даёт «предзагрузку» — данные начинают грузиться чуть раньше, чем пользователь доскроллит до самого конца.

Можешь копировать оба файла. Если что-то нужно подправить (например, текст «Все записи загружены», отступы, или поведение при ошибке сети) — говори.
