<template>
  <div class="ww-datagrid" :class="{ editing: isEditing }" :style="cssVars">
    <ag-grid-vue :rowData="computedRowData" :columnDefs="columnDefs" :defaultColDef="defaultColDef"
      :domLayout="content.layout === 'auto' ? 'autoHeight' : 'normal'" :style="style" :rowSelection="rowSelection"
      :selection-column-def="{ pinned: true }" :theme="theme" :getRowId="getRowId" :pagination="content.pagination"
      :paginationPageSize="content.paginationPageSize || 10" :paginationPageSizeSelector="false"
      :suppressMovableColumns="!content.movableColumns" :pinnedBottomRowData="pinnedBottomRowData"
      :columnHoverHighlight="content.columnHoverHighlight" :locale-text="localeText" @grid-ready="onGridReady"
      @row-selected="onRowSelected" @selection-changed="onSelectionChanged" @cell-value-changed="onCellValueChanged"
      @filter-changed="onFilterChanged" @sort-changed="onSortChanged" @row-double-clicked="onRowDoubleClicked">
    </ag-grid-vue>
  </div>
</template>

<script>
import { shallowRef, watchEffect, computed, watch } from "vue";
import { AgGridVue } from "ag-grid-vue3";
import {
  AllCommunityModule,
  ModuleRegistry,
  themeQuartz,
} from "ag-grid-community";
import {
  AG_GRID_LOCALE_EN,
  AG_GRID_LOCALE_FR,
  AG_GRID_LOCALE_DE,
  AG_GRID_LOCALE_ES,
  AG_GRID_LOCALE_PT,
} from "@ag-grid-community/locale";
import ActionCellRenderer from "./components/ActionCellRenderer.vue";
import ImageCellRenderer from "./components/ImageCellRenderer.vue";
import WewebCellRenderer from "./components/WewebCellRenderer.vue";
import ComparativeCellRenderer from "./components/ComparativeCellRenderer.vue";
import SkeletonCellRenderer from "./components/SkeletonCellRenderer.vue";

console.log("AG Grid version:", AG_GRID_LOCALE_FR);

// TODO: maybe register less modules
// TODO: maybe register modules per grid instead of globally
ModuleRegistry.registerModules([AllCommunityModule]);

export default {
  components: {
    AgGridVue,
    ActionCellRenderer,
    ImageCellRenderer,
    WewebCellRenderer,
    ComparativeCellRenderer,
    SkeletonCellRenderer,
  },
  props: {
    content: {
      type: Object,
      required: true,
    },
    uid: {
      type: String,
      required: true,
    },
    /* wwEditor:start */
    wwEditorState: { type: Object, required: true },
    /* wwEditor:end */
  },
  emits: ["trigger-event", "update:content:effect"],
  setup(props, ctx) {
    const { resolveMappingFormula } = wwLib.wwFormula.useFormula();

    const gridApi = shallowRef(null);
    const { value: selectedRows, setValue: setSelectedRows } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "selectedRows",
        type: "array",
        defaultValue: [],
        readonly: true,
      });
    const { value: filterValue, setValue: setFilters } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "filters",
        type: "object",
        defaultValue: {},
        readonly: true,
      });
    const { value: sortValue, setValue: setSort } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "sort",
        type: "object",
        defaultValue: {},
        readonly: true,
      });
    const { value: dataValue, setValue: setData } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "data",
        type: "array",
        defaultValue: [],
        readonly: true,
      });

    watchEffect(() => {
      if (!gridApi.value) return;
      if (props.content.initialFilters) {
        gridApi.value.setFilterModel(props.content.initialFilters);
      }
      if (props.content.initialSort) {
        gridApi.value.applyColumnState({
          state: props.content.initialSort || [],
          defaultState: { sort: null },
        });
      }
    });

    const onGridReady = (params) => {
      gridApi.value = params.api;
      updateDataAfterFilterAndSort();
    };

    const sizeColumnsToFit = () => {
      const allColumnIds = [];
      gridApi.value.getColumns().forEach((column) => {
        allColumnIds.push(column.getId());
      });
      gridApi.value.autoSizeColumns(allColumnIds, false);
    };

    const updateDataAfterFilterAndSort = () => {
      if (!gridApi.value) return;

      const rowsData = [];
      gridApi.value.forEachNodeAfterFilterAndSort((node) => {
        if (node.data) {
          rowsData.push(node.data);
        }
      });

      setData(rowsData);
    };

    watch(
      () => props.content.rowData,
      () => {
        // Quando content.rowData mudar, atualize imediatamente a variável data
        const rawData = wwLib.wwUtils.getDataFromCollection(props.content.rowData);
        setData(Array.isArray(rawData) ? rawData : []);
      },
      { immediate: true } // Executa imediatamente ao inicializar o componente
    );

    const onRowSelected = (event) => {
      const name = event.node.isSelected() ? "rowSelected" : "rowDeselected";
      ctx.emit("trigger-event", {
        name,
        event: { row: event.data },
      });
    };

    const onSelectionChanged = (event) => {
      if (!gridApi.value) return;
      const selected = gridApi.value.getSelectedRows() || [];
      setSelectedRows(selected);
    };

    const onFilterChanged = (event) => {
      if (!gridApi.value) return;
      const filterModel = gridApi.value.getFilterModel();
      if (
        JSON.stringify(filterModel || {}) !==
        JSON.stringify(filterValue.value || {})
      ) {
        setFilters(filterModel);
        ctx.emit("trigger-event", {
          name: "filterChanged",
          event: filterModel,
        });
      }
      updateDataAfterFilterAndSort();
    };

    const onSortChanged = (event) => {
      if (!gridApi.value) return;
      const state = gridApi.value.getState();
      if (
        JSON.stringify(state.sort?.sortModel || []) !==
        JSON.stringify(sortValue.value || [])
      ) {
        setSort(state.sort?.sortModel || []);
        ctx.emit("trigger-event", {
          name: "sortChanged",
          event: state.sort?.sortModel || [],
        });
      }
      updateDataAfterFilterAndSort();
    };

    /* wwEditor:start */
    const { createElement } = wwLib.useCreateElement();
    /* wwEditor:end */

    return {
      resolveMappingFormula,
      onGridReady,
      onRowSelected,
      onSelectionChanged,
      sizeColumnsToFit,
      updateDataAfterFilterAndSort,
      gridApi,
      onFilterChanged,
      onSortChanged,
      localeText: computed(() => {
        switch (props.content.lang) {
          case "fr":
            return AG_GRID_LOCALE_FR;
          case "de":
            return AG_GRID_LOCALE_DE;
          case "es":
            return AG_GRID_LOCALE_ES;
          case "pt":
            return AG_GRID_LOCALE_PT;
          case "custom":
            return {
              ...AG_GRID_LOCALE_EN,
              ...(props.content.localeText || {}),
            };
          default:
            AG_GRID_LOCALE_EN;
        }
      }),
      /* wwEditor:start */
      createElement,
      /* wwEditor:end */
    };
  },
  computed: {
    rowData() {
      const data = wwLib.wwUtils.getDataFromCollection(this.content.rowData);
      return Array.isArray(data) ? data ?? [] : [];
    },
    computedRowData() {
      if (this.content.loading) {
        // Gerar 15 linhas de skeleton (ou o número desejado)
        const skeletonRows = Array(15)
          .fill()
          .map((_, index) => {
            const row = {
              _isSkeletonRow: true,
              _skeletonId: `skeleton-${index}` // Propriedade para uso no idFormula caso necessário
            };
            // Adicionar um campo para cada coluna
            this.content.columns.forEach(column => {
              if (column.field) {
                row[column.field] = null;
              }
            });
            return row;
          });
        return skeletonRows;
      }

      // Retornar os dados originais quando não estiver carregando
      return this.rowData;
    },
    defaultColDef() {
      return {
        editable: false,
        resizable: this.content.resizableColumns,
      };
    },
    pinnedBottomRowData() {
      if (!this.content.showTotalRow) return null;

      const totalRow = {};

      this.content.columns.forEach(column => {
        if (column.field) {
          totalRow[column.field] = column.totalValue !== undefined ? column.totalValue : '';
        }
      });

      return [totalRow];
    },
    columnDefs() {
      const isLoading = this.content.loading;

      // Primeiro, processamos as colunas normalmente
      const processedColumns = this.content.columns.map((col) => {
        const minWidth =
          !col.minWidth || col.minWidth === "auto"
            ? null
            : wwLib.wwUtils.getLengthUnit(col.minWidth)?.[0];
        const maxWidth =
          !col.maxWidth || col.maxWidth === "auto"
            ? null
            : wwLib.wwUtils.getLengthUnit(col.maxWidth)?.[0];
        const width =
          !col.width || col.width === "auto" || col.widthAlgo === "flex"
            ? null
            : wwLib.wwUtils.getLengthUnit(col.width)?.[0];
        const flex = col.widthAlgo === "flex" ? col.flex ?? 1 : null;
        const commonProperties = {
          minWidth,
          maxWidth,
          pinned: col.pinned === "none" ? false : col.pinned,
          width,
          flex,
          hide: !!col.hide,
        };

        let columnDef;

        if (isLoading) {
          columnDef = {
            ...commonProperties,
            headerName: col.headerName,
            field: col.field,
            cellRenderer: "SkeletonCellRenderer",
            sortable: false,
            filter: false,
          };
        } else {
          switch (col.cellDataType) {
            case "action": {
              columnDef = {
                ...commonProperties,
                headerName: col.headerName,
                cellRenderer: "ActionCellRenderer",
                cellRendererParams: {
                  name: col.actionName,
                  label: col.actionLabel,
                  trigger: this.onActionTrigger,
                  withFont: !!this.content.actionFont,
                },
                sortable: false,
                filter: false,
              };
              break;
            }
            case "custom":
              columnDef = {
                ...commonProperties,
                headerName: col.headerName,
                field: col.field,
                cellRenderer: "WewebCellRenderer",
                cellRendererParams: {
                  containerId: col.containerId,
                },
                sortable: col.sortable,
                filter: col.filter,
              };
              break;
            case "image": {
              columnDef = {
                ...commonProperties,
                headerName: col.headerName,
                field: col.field,
                cellRenderer: "ImageCellRenderer",
                cellRendererParams: {
                  width: col.imageWidth,
                  height: col.imageHeight,
                },
              };
              break;
            }
            case "formatted-number": {
              columnDef = {
                ...commonProperties,
                headerName: col.headerName,
                field: col.field,
                sortable: col.sortable,
                filter: col.filter,
                editable: col.editable,
                valueFormatter: (params) => {
                  if (params.value === null || params.value === undefined) return '';
                  return Number(params.value).toLocaleString('pt-BR', {
                    minimumFractionDigits: col.decimalPlaces || 2,
                    maximumFractionDigits: col.decimalPlaces || 2
                  });
                }
              };

              if (col.comparative) {
                columnDef.cellRenderer = "ComparativeCellRenderer";
              }

              break;
            }
            case "currency": {
              columnDef = {
                ...commonProperties,
                headerName: col.headerName,
                field: col.field,
                sortable: col.sortable,
                filter: col.filter,
                editable: col.editable,
                valueFormatter: (params) => {
                  if (params.value === null || params.value === undefined) return '';
                  return `R$ ${Number(params.value).toLocaleString('pt-BR', {
                    minimumFractionDigits: col.decimalPlaces || 2,
                    maximumFractionDigits: col.decimalPlaces || 2
                  })}`;
                }
              };

              if (col.comparative) {
                columnDef.cellRenderer = "ComparativeCellRenderer";
              }

              break;
            }
            case "percentage": {
              columnDef = {
                ...commonProperties,
                headerName: col.headerName,
                field: col.field,
                sortable: col.sortable,
                filter: col.filter,
                editable: col.editable,
                valueFormatter: (params) => {
                  if (params.value === null || params.value === undefined) return '';
                  return `${Number(params.value).toLocaleString('pt-BR', {
                    minimumFractionDigits: 2,
                    maximumFractionDigits: 2
                  })}%`;
                }
              };

              if (col.comparative) {
                columnDef.cellRenderer = "ComparativeCellRenderer";
              }

              break;
            }
            default: {
              columnDef = {
                ...commonProperties,
                headerName: col.headerName,
                field: col.field,
                sortable: col.sortable,
                filter: col.filter,
                editable: col.editable,
              };

              if (col.comparative) {
                columnDef.cellRenderer = "ComparativeCellRenderer";
              }

              if (col.useCustomLabel) {
                columnDef.valueFormatter = (params) => {
                  return this.resolveMappingFormula(
                    col.displayLabelFormula,
                    params.value
                  );
                };
              }
            }
          }
        }

        // Adicionamos informação sobre o grupo da coluna para processamento posterior
        return {
          columnDef,
          columnGroup: col.columnGroup
        };
      });

      // Agora vamos organizar as colunas em grupos
      const columnGroups = {};
      const finalColumnDefs = [];

      // Percorremos as colunas processadas para organizá-las
      processedColumns.forEach(({ columnDef, columnGroup }) => {
        // Se a coluna não pertence a nenhum grupo, adicionamos diretamente ao array final
        if (columnGroup === "") {
          finalColumnDefs.push(columnDef);
          return;
        }

        // Se o grupo já existe, adicionamos a coluna aos filhos
        if (columnGroups[columnGroup]) {
          columnGroups[columnGroup].children.push(columnDef);
        }
        // Se o grupo não existe, criamos um novo
        else {
          columnGroups[columnGroup] = {
            headerName: columnGroup,
            children: [columnDef]
          };
          // Adicionamos o novo grupo ao array final
          finalColumnDefs.push(columnGroups[columnGroup]);
        }
      });

      return finalColumnDefs;
    },
    rowSelection() {
      if (this.content.rowSelection === "multiple") {
        return {
          mode: "multiRow",
          checkboxes: !this.content.disableCheckboxes,
          headerCheckbox: !this.content.disableCheckboxes,
          selectAll: this.content.selectAll || "all",
          enableClickSelection: this.content.enableClickSelection,
        };
      } else if (this.content.rowSelection === "single") {
        return {
          mode: "singleRow",
          checkboxes: !this.content.disableCheckboxes,
          enableClickSelection: this.content.enableClickSelection,
        };
      } else {
        return {
          mode: "singleRow",
          checkboxes: false,
          isRowSelectable: () => false,
          enableClickSelection: this.content.enableClickSelection,
        };
      }
    },
    style() {
      if (this.content.layout === "auto") return {};
      return {
        height: this.content.height || "400px",
      };
    },
    cssVars() {
      return {
        "--ww-data-grid_action-backgroundColor":
          this.content.actionBackgroundColor,
        "--ww-data-grid_action-color": this.content.actionColor,
        "--ww-data-grid_action-padding": this.content.actionPadding,
        "--ww-data-grid_action-border": this.content.actionBorder,
        "--ww-data-grid_action-borderRadius": this.content.actionBorderRadius,
        ...(this.content.actionFont
          ? { "--ww-data-grid_action-font": this.content.actionFont }
          : {
            "--ww-data-grid_action-fontSize": this.content.actionFontSize,
            "--ww-data-grid_action-fontFamily": this.content.actionFontFamily,
            "--ww-data-grid_action-fontWeight": this.content.actionFontWeight,
            "--ww-data-grid_action-fontStyle": this.content.actionFontStyle,
            "--ww-data-grid_action-lineHeight": this.content.actionLineHeight,
          }),
      };
    },
    theme() {
      return themeQuartz.withParams({
        headerBackgroundColor: this.content.headerBackgroundColor,
        headerTextColor: this.content.headerTextColor,
        headerFontSize: this.content.headerFontSize,
        headerFontFamily: this.content.headerFontFamily,
        headerFontWeight: this.content.headerFontWeight,
        borderColor: this.content.borderColor,
        cellTextColor: this.content.cellColor,
        cellFontFamily: this.content.cellFontFamily,
        dataFontSize: this.content.cellFontSize,
        oddRowBackgroundColor: this.content.rowAlternateColor,
        backgroundColor: this.content.rowBackgroundColor,
        rowHoverColor: this.content.rowHoverColor,
        selectedRowBackgroundColor: this.content.selectedRowBackgroundColor,
        rowVerticalPaddingScale: this.content.rowVerticalPaddingScale || 1,
        menuBackgroundColor: this.content.menuBackgroundColor,
        menuTextColor: this.content.menuTextColor,
        columnHoverColor: this.content.columnHoverColor,
        foregroundColor: this.content.textColor,
        checkboxCheckedBackgroundColor: this.content.selectionCheckboxColor,
        rangeSelectionBorderColor: this.content.cellSelectionBorderColor,
        checkboxUncheckedBorderColor: this.content.checkboxUncheckedBorderColor,
        focusShadow: this.content.focusShadow?.length
          ? this.content.focusShadow
          : undefined,
      });
    },
    isEditing() {
      /* wwEditor:start */
      return (
        this.wwEditorState.editMode === wwLib.wwEditorHelper.EDIT_MODES.EDITION
      );
      /* wwEditor:end */
      // eslint-disable-next-line no-unreachable
      return false;
    },
  },
  methods: {
    getRowId(params) {
      if (params.data?._isSkeletonRow) {
        return params.data._skeletonId;
      }
      return this.resolveMappingFormula(this.content.idFormula, params.data);
    },
    onActionTrigger(event) {
      this.$emit("trigger-event", {
        name: "action",
        event,
      });
    },
    onCellValueChanged(event) {
      this.$emit("trigger-event", {
        name: "cellValueChanged",
        event: {
          oldValue: event.oldValue,
          newValue: event.newValue,
          columnId: event.column.getColId(),
          row: event.data,
        },
      });
    },
    onRowDoubleClicked(event) {
      this.$emit("trigger-event", {
        name: "rowDoubleClicked",
        event: {
          row: event.data,
          id: event.node.id,
          index: event.node.sourceRowIndex,
          displayIndex: event.rowIndex,
        },
      });
    },
    /* wwEditor:start */
    generateColumns() {
      this.$emit("update:content", {
        columns: this.rowData?.[0]
          ? Object.keys(this.rowData[0]).map((key) => ({
            field: key,
            sortable: true,
            filter: true,
          }))
          : [],
      });
    },
    getOnActionTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        actionName: "actionName",
        row: data[0],
        id: 0,
        index: 0,
        displayIndex: 0,
      };
    },
    getOnCellValueChangedTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        oldValue: "oldValue",
        newValue: "newValue",
        columnId: "columnId",
        row: data[0],
      };
    },
    getSelectionTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        row: data[0],
      };
    },
    getRowClickedTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        row: data[0],
        id: 0,
        index: 0,
        displayIndex: 0,
      };
    },
    /* wwEditor:end */
  },
  /* wwEditor:start */
  watch: {
    columnDefs: {
      async handler() {
        if (this.wwEditorState?.boundProps?.columns) return;
        this.gridApi.resetColumnState();

        if (this.wwEditorState.isACopy) return;

        // We assume there will only be one custom column each time
        const columnIndex = (this.content.columns || []).findIndex(
          (col) => col.cellDataType === "custom" && !col.containerId
        );
        if (columnIndex === -1) return;
        const newColumns = [...this.content.columns];
        let column = { ...newColumns[columnIndex] };
        column.containerId = await this.createElement("ww-flexbox", {
          _state: { name: `Cell ${column.headerName || column.field}` },
        });
        newColumns[columnIndex] = column;
        this.$emit("update:content:effect", { columns: newColumns });
      },
      deep: true,
    },
  },
  /* wwEditor:end */
};
</script>

<style lang="scss">
.ww-datagrid {
  position: relative;

  /* wwEditor:start */
  &.editing {
    &::before {
      content: "";
      position: absolute;
      inset: 0;
      display: block;
      pointer-events: initial;
      z-index: 10;
    }
  }

  /* wwEditor:end */
}

.ag-cell {
  font-weight: 600;
}
</style>
