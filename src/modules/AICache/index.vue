/**
* Copyright(c) 2026 The Rainway AI Gateway (壬远AI网关) Authors.
*
* Licensed under the Apache License, Version 2.0 (the "License");
* you may not use this file except in compliance with the License.
* You may obtain a copy of the License at
*
* http: //www.apache.org/licenses/LICENSE-2.0
*
* Unless required by applicable law or agreed to in writing, software
* distributed under the License is distributed on an "AS IS" BASIS,
* WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
* See the License for the specific language governing permissions and
* limitations under the License.
*/
<template>
  <div class="ai-cache">
    <div class="route-editor-header">
      <div class="route-owner-label">
        {{ $t('aiCache.pageTitle') }}
      </div>
      <div class="route-header-actions">
        <Button
          v-show="mode === 'edit'"
          size="small"
          type="primary"
          @click="onAddRule"
        >
          {{ $t('aiCache.addRule') }}
        </Button>
      </div>
      <div class="route-header-submit">
        <template v-if="mode === 'view'">
          <Button size="small" type="primary" @click="enterEditMode">
            {{ $t('route.enterEditMode') }}
          </Button>
        </template>
        <template v-else>
          <Button size="small" @click="exitEditMode">
            {{ $t('route.exitEditMode') }}
          </Button>
          <Button
            size="small"
            type="success"
            :loading="submitting"
            @click="submitRules"
          >
            {{ $t('com.submitAndEffect') }}
          </Button>
        </template>
      </div>
    </div>

    <pageTable
      ref="ruleTable"
      :columns="ruleColumns"
      :tableData="rules"
      :loading="loading"
    />

    <Drawer
      v-model="viewVisible"
      :title="$t('aiCache.viewTitle')"
      :width="60"
      :mask-closable="true"
      @on-close="currentRule = null"
    >
      <RuleView
        v-if="viewVisible && currentRule"
        :rule="currentRule"
      />
    </Drawer>

    <Drawer
      v-model="formVisible"
      :title="formTitle"
      :width="60"
      :mask-closable="false"
    >
      <RuleForm
        v-if="formVisible"
        :rule="currentRule"
        @submit="onFormSubmit"
      />
    </Drawer>
  </div>
</template>

<script>
import pageTable from '@/components/table/pageTable';
import RuleForm from './components/RuleForm.vue';
import RuleView from './components/RuleView.vue';
import { cloneDeep } from 'lodash';

export default {
  name: 'AICache',

  components: {
    pageTable,
    RuleForm,
    RuleView
  },

  data() {
    const that = this;
    return {
      loading: false,
      submitting: false,
      rules: [],
      originalRows: null,
      mode: 'view',
      viewVisible: false,
      formVisible: false,
      formMode: 'add',
      currentRule: null,
      currentRuleIndex: -1,
      ruleColumns: that.buildRuleColumns()
    };
  },

  computed: {
    isDirty() {
      if (this.mode !== 'edit' || this.originalRows === null) {
        return false;
      }
      return JSON.stringify(this.rules) !== JSON.stringify(this.originalRows);
    },

    formTitle() {
      return this.formMode === 'edit'
        ? this.$t('aiCache.editTitle')
        : this.$t('aiCache.addTitle');
    }
  },

  watch: {
    mode() {
      this.ruleColumns = this.buildRuleColumns();
    }
  },

  methods: {
    buildRuleColumns() {
      const that = this;
      return [
        {
          title: that.$t('route.ruleName'),
          key: 'name',
          minWidth: 160,
          sortable: 'custom',
          searchable: true,
          render(h, params) {
            return h('span', params.row.name || '-');
          }
        },
        {
          title: that.$t('route.expression'),
          key: 'cond',
          minWidth: 260,
          sortable: 'custom',
          searchable: true,
          render(h, params) {
            return h('span', params.row.cond || '-');
          }
        },
        {
          title: that.$t('aiCache.colStrategy'),
          key: 'cache_key_strategy',
          minWidth: 180,
          sortable: 'custom',
          searchable: true,
          searchType: 'select',
          searchFilters: [
            { label: 'lastQuestion', value: 'lastQuestion' },
            { label: 'allQuestions', value: 'allQuestions' },
            { label: 'disabled', value: 'disabled' }
          ],
          render(h, params) {
            const strategy = params.row.cache_key_strategy;
            const label = that.strategyLabel(strategy);
            return h(
              'Tag',
              { props: { color: strategy === 'disabled' ? 'warning' : 'primary' } },
              label
            );
          }
        },
        {
          title: that.$t('aiCache.colTtl'),
          key: 'cache_ttl',
          minWidth: 130,
          sortable: 'custom',
          render(h, params) {
            return h('span', that.formatTtl(params.row.cache_ttl));
          }
        },
        {
          title: that.$t('aiCache.colMaxBody'),
          key: 'max_body_bytes',
          minWidth: 130,
          sortable: 'custom',
          render(h, params) {
            return h('span', that.formatBytes(params.row.max_body_bytes));
          }
        },
        {
          title: that.$t('aiCache.colMaxValue'),
          key: 'max_value_bytes',
          minWidth: 130,
          sortable: 'custom',
          render(h, params) {
            return h('span', that.formatBytes(params.row.max_value_bytes));
          }
        },
        {
          title: that.$t('com.operation'),
          key: 'action',
          width: 220,
          render(h, params) {
            const row = params.row;
            if (that.mode === 'view') {
              return h(
                'Button',
                {
                  props: { size: 'small', type: 'primary' },
                  on: { click: () => that.onViewRule(row) }
                },
                that.$t('com.see')
              );
            }
            return h('div', [
              h(
                'Button',
                {
                  props: { size: 'small', type: 'primary' },
                  style: { marginRight: '8px' },
                  on: { click: () => that.onEditRule(row) }
                },
                that.$t('com.edit')
              ),
              h(
                'Button',
                {
                  props: { size: 'small', type: 'error' },
                  on: { click: () => that.onDeleteRule(row) }
                },
                that.$t('com.del')
              )
            ]);
          }
        }
      ];
    },

    strategyLabel(strategy) {
      const map = {
        lastQuestion: this.$t('aiCache.strategyLastQuestion'),
        allQuestions: this.$t('aiCache.strategyAllQuestions'),
        disabled: this.$t('aiCache.strategyDisabled')
      };
      return map[strategy] || strategy || '-';
    },

    formatTtl(value) {
      if (value === null || value === undefined) {
        return '-';
      }
      return Number(value) === 0
        ? this.$t('aiCache.ttlNever')
        : this.$t('aiCache.ttlSecondsText', { value });
    },

    formatBytes(value) {
      if (value === null || value === undefined) {
        return '-';
      }
      return this.$t('aiCache.bytesText', { value });
    },

    fetchRules() {
      this.loading = true;
      this.$request({
        url: 'ai-cache-rules',
        method: 'get',
        openapi: true,
        unneedTips: true
      })
        .then(res => {
          if (res && res.status === 200) {
            const data = res.data.Data || {};
            this.rules = data.rules || [];
          } else {
            this.$Message.error(this.$t('aiCache.loadFailed'));
          }
        })
        .catch(() => {
          this.$Message.error(this.$t('aiCache.loadFailed'));
        })
        .finally(() => {
          this.loading = false;
        });
    },

    findRowIndex(row) {
      if (!row) {
        return -1;
      }
      return this.rules.findIndex(item => item.name === row.name);
    },

    enterEditMode() {
      this.mode = 'edit';
      this.originalRows = cloneDeep(this.rules);
    },

    exitEditMode() {
      if (this.isDirty) {
        this.$Modal.confirm({
          title: this.$t('com.informationTips'),
          content: this.$t('aiCache.exitConfirm'),
          onOk: () => {
            this.performExitEditMode();
            this.$Message.info(this.$t('aiCache.exitDone'));
          },
          onCancel: () => {
            this.$Message.info(this.$t('aiCache.exitCancelled'));
          }
        });
        return;
      }
      this.performExitEditMode();
    },

    performExitEditMode() {
      this.mode = 'view';
      if (this.originalRows) {
        this.rules = cloneDeep(this.originalRows);
      }
      this.originalRows = null;
      this.fetchRules();
    },

    onAddRule() {
      this.currentRuleIndex = -1;
      this.currentRule = null;
      this.formMode = 'add';
      this.formVisible = true;
    },

    onEditRule(row) {
      const index = this.findRowIndex(row);
      if (index < 0) {
        return;
      }
      this.currentRuleIndex = index;
      this.currentRule = cloneDeep(this.rules[index]);
      this.formMode = 'edit';
      this.formVisible = true;
    },

    onViewRule(row) {
      const index = this.findRowIndex(row);
      if (index < 0) {
        return;
      }
      this.currentRule = cloneDeep(this.rules[index]);
      this.viewVisible = true;
    },

    onDeleteRule(row) {
      this.$Modal.confirm({
        title: this.$t('com.informationTips'),
        content: this.$t('aiCache.deleteConfirm'),
        onOk: () => {
          const index = this.findRowIndex(row);
          if (index >= 0) {
            this.rules.splice(index, 1);
          }
          this.$Message.success(this.$t('com.tipDelSucc'));
        },
        onCancel: () => {
          this.$Message.info(this.$t('com.tipCancelDel'));
        }
      });
    },

    onFormSubmit(data) {
      const excludeIndex = this.formMode === 'edit' ? this.currentRuleIndex : -1;
      const duplicated = this.rules.some((rule, index) => {
        if (index === excludeIndex) {
          return false;
        }
        return rule.name === data.name;
      });
      if (duplicated) {
        this.$Message.error(this.$t('aiCache.tipNameDuplicate'));
        return;
      }

      if (this.formMode === 'add') {
        this.rules.unshift(data);
      } else if (this.currentRuleIndex >= 0) {
        this.rules.splice(this.currentRuleIndex, 1, data);
      }
      this.formVisible = false;
      this.$Message.success(
        this.formMode === 'add'
          ? this.$t('aiCache.addSuccess')
          : this.$t('aiCache.saveSuccess')
      );
    },

    submitRules() {
      const names = (this.rules || []).map(rule => rule.name).filter(Boolean);
      const uniqueNames = [...new Set(names)];
      if (names.length !== uniqueNames.length) {
        this.$Message.error(this.$t('aiCache.tipNameDuplicate'));
        return;
      }

      this.submitting = true;
      this.$request({
        url: 'ai-cache-rules',
        method: 'put',
        data: { rules: this.buildPayload() },
        openapi: true
      })
        .then(res => {
          if (res && res.status === 200) {
            this.$Message.success(this.$t('aiCache.submitSuccess'));
            this.originalRows = null;
            this.mode = 'view';
            this.fetchRules();
          } else {
            this.$Message.error(this.$t('com.tipSubmitFailed'));
          }
        })
        .catch(() => {
          this.$Message.error(this.$t('com.tipSubmitFailed'));
        })
        .finally(() => {
          this.submitting = false;
        });
    },

    buildPayload() {
      return this.rules.map(rule => {
        const clean = cloneDeep(rule);
        delete clean.created_at;
        delete clean.updated_at;
        return clean;
      });
    }
  },

  mounted() {
    this.fetchRules();
  }
};
</script>

<style lang="less" scoped>
.ai-cache {
  .route-editor-header {
    position: relative;
    margin-bottom: 16px;
  }

  .route-owner-label {
    color: #515a6e;
    font-size: 14px;
    margin-bottom: 12px;
  }

  .route-header-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .route-header-submit {
    position: absolute;
    top: 0;
    right: 0;
    display: flex;
    align-items: center;
    gap: 8px;
  }
}
</style>