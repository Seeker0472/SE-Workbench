<template>
  <tiny-grid
    ref="grid"
    :pager="pagerConfig"
    :fetch-data="fetchData"
    :edit-config="
      rolePermission.includes('i18n::update')
        ? { trigger: 'click', mode: 'cell', showStatus: true }
        : undefined
    "
    :loading="loading"
    remote-filter
    refresh
    @edit-closed="onEditClosed"
  >
    <template #toolbar>
      <tiny-grid-toolbar refresh></tiny-grid-toolbar>
    </template>
    <tiny-grid-column field="id" title="id" width="60"></tiny-grid-column>
    <tiny-grid-column
      field="key"
      title="key"
      :editor="{ component: 'input', autoselect: true }"
      :filter="keyFilter"
    ></tiny-grid-column>
    <tiny-grid-column
      field="content"
      title="content"
      :editor="{ component: 'input' }"
      :filter="contentFilter"
    ></tiny-grid-column>
    <tiny-grid-column
      field="lang"
      title="lang"
      :editor="{ component: 'select', options }"
      :format-config="{ async: true, data: options, type: 'enum' }"
      :filter="langFilter"
    ></tiny-grid-column>
    <tiny-grid-column>
      <template #default="data">
        <tiny-button
          v-permission="'i18n::remove'"
          @click="removeLocale(data.row)"
        >
          {{ $t('locale.remove') }}
        </tiny-button>
      </template>
    </tiny-grid-column>
  </tiny-grid>
</template>

<script lang="ts" setup>
  import { getAllLocalItems, patchLocal, deleteLocale } from '@/api/local';
  import useLoading from '@/hooks/loading';
  import { useUserStore } from '@/store';
  import { useLocales } from '@/store/modules/locales';
  import { FilterType, InputFilterValue } from '@/types/global';
  import {
    Notify,
    Grid as TinyGrid,
    GridColumn as TinyGridColumn,
    Button as TinyButton,
    GridToolbar as TinyGridToolbar,
    TinyModal,
  } from '@opentiny/vue';
  import { computed, ref } from 'vue';

  const grid = ref();
  const localeStore = useLocales();

  export type LocalTableData = {
    id: number;
    key: string;
    content: string;
    lang: string;
  };
  const userStore = useUserStore();
  const rolePermission = computed(() => userStore.rolePermission);

  const keyFilter = {
    inputFilter: true,
  };
  const contentFilter = {
    inputFilter: true,
  };
  const langFilter = {
    enumable: true,
    multi: true,
    values: () => {
      return Promise.resolve(
        localeStore.lang.map((language) => ({
          label: language.name,
          value: language.id,
        })),
      );
    },
  };
  const pagerConfig = ref({
    attrs: {
      currentPage: 1,
      pageSize: 5,
      pageSizes: Array.from({ length: 20 }).map(
        (cur, index) => (index + 1) * 5,
      ),
      total: 0,
      align: 'right',
      layout: 'total, prev, pager, next, jumper, sizes',
    },
  });

  const options = computed(() =>
    localeStore.lang.map((language) => ({
      label: language.name,
      value: language.name,
    })),
  );
  const { loading, setLoading } = useLoading();
  if (!localeStore.lang.length) {
    localeStore.fetchLang();
  }

  let currentPage = 0;

  const filterInputValue2String = (value: InputFilterValue) => {
    let str = '';
    if (value.relation === 'contains') {
      str += '%';
    }
    str += value.text;
    if (value.relation === 'startwith' || value.relation === 'contains') {
      str += '%';
    }
    return str;
  };

  const getData = ({
    page,
    filters,
  }: {
    page: { pageSize: number; currentPage: number };
    filters: FilterType;
  }) => {
    const key = filters.key
      ? filterInputValue2String(filters.key.value as InputFilterValue)
      : undefined;
    const content = filters.content
      ? filterInputValue2String(filters.content.value as InputFilterValue)
      : undefined;
    const lang =
      filters.lang && (filters.lang.value as number[]).length
        ? (filters.lang.value as number[]).toString()
        : undefined;
    const { pageSize } = page;
    currentPage = page.currentPage;
    setLoading(true);
    return new Promise((resolve) => {
      getAllLocalItems(currentPage, pageSize, 0, { key, content, lang })
        .then(({ data }) => {
          resolve({
            result: data.items.map((item) => {
              return {
                id: item.id,
                key: item.key,
                content: item.content,
                lang: item.lang.name,
              };
            }),
            page: {
              total: data.meta.totalItems,
            },
          });
        })
        .finally(() => {
          setLoading(false);
        });
    });
  };
  const onEditClosed = ({ row }: { row: Record<string, any> }) => {
    if (grid.value.hasRowChange(row)) {
      const langId = localeStore.lang.filter(
        (lang) => lang.name === row.lang,
      )[0].id;
      patchLocal(row.id, {
        content: row.content,
        key: row.key,
        lang: langId,
      })
        .then(() => {
          Notify({
            type: 'info',
            message: '更新成功',
          });
        })
        .catch((error) => {
          grid.value.revertData(row);
          if (error.response && error.response.data) {
            const errorMessage = error.response.data.message || '未知错误';
            TinyModal.message({
              message: errorMessage,
              status: 'error',
            });
          }
        })
        .finally(() => {
          setLoading(false);
        });
    }
  };
  const removeLocale = (row: any) => {
    setLoading(true);
    deleteLocale(row.id)
      .then(() => {
        localeStore.$patch({
          locales: localeStore.locales.filter((locale) => locale.id !== row.id),
        });
        grid.value.remove(row);
      })
      .catch((error) => {
        if (error.response && error.response.data) {
          const errorMessage = error.response.data.message || '未知错误';
          TinyModal.message({
            message: errorMessage,
            status: 'error',
          });
        }
      })
      .finally(() => {
        setLoading(false);
      });
  };

  const fetchData = ref({
    api: getData,
    filter: true,
  });
  defineExpose({
    reload: () => {
      grid.value.handleFetch();
    },
  });
</script>
