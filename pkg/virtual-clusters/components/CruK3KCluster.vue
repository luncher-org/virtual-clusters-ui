<script>
import { mapGetters } from 'vuex';

import LabeledSelect from '@shell/components/form/LabeledSelect';
import NameNsDescription from '@shell/components/form/NameNsDescription';
import Tabbed from '@shell/components/Tabbed';
import Tab from '@shell/components/Tabbed/Tab';
import CruResource from '@shell/components/CruResource';
import Loading from '@shell/components/Loading';
import Labels from '@shell/edit/provisioning.cattle.io.cluster/Labels';
import KeyValue from '@shell/components/form/KeyValue.vue';
import ArrayList from '@shell/components/form/ArrayList';

import { Banner } from '@components/Banner';
import { saferDump } from '@shell/utils/create-yaml';
import LabeledInput from '@components/Form/LabeledInput/LabeledInput.vue';
import RadioGroup from '@components/Form/Radio/RadioGroup.vue';

import CreateEditView from '@shell/mixins/create-edit-view';

import ClusterMembershipEditor, { canViewClusterMembershipEditor } from '@shell/components/form/Members/ClusterMembershipEditor';
import { CAPI, MANAGEMENT } from '@shell/config/types';
import { K3K } from '../types';
import ClusterAppearance from '@shell/components/form/ClusterAppearance';
import HostCluster from './HostCluster.vue';
import { _CREATE } from '@shell/config/query-params';
import cloneDeep from 'lodash/cloneDeep';
import { allHash } from '@shell/utils/promise';
import importConfigMapTemplate from '../resources/import-configmap.json';
import importJobTemplate from '../resources/import-job.json';

const defaultCluster = {
  type:       K3K.CLUSTER,
  apiVersion: 'k3k.io/v1alpha1',
  kind:       'Cluster',
  metadata:   { name: '' },
  spec:       {
    mode:        'shared',
    agents:      0,
    persistence: {
      storageRequestSize: '1G', type: 'dynamic', storageClassName: 'local-path'
    },
    servers: 1,
  }
};

/**
 * provisioning.cattle.io.cluster default annotations
 *
 * also set before creation:
 * 'ui.rancher/parent-cluster' - host cluster's norman cluster id
 * 'ui.rancher/parent-cluster-display' - host cluster provisioning cluster displayName model property
 * 'ui.rancher/k3k-namespace'  - namespace/id of the k3k cluster in the host cluster
 */
const defaultAnnotations = {
  // prevent k3s-upgrade-controller from running: this will be managed by k3k
  'rancher.io/imported-cluster-version-management': 'false',
  // display machine provider in cluster mgmt list
  'ui.rancher/provider':                            'k3k'
};

export default {
  emits: ['update:value'],

  components: {
    LabeledSelect,
    NameNsDescription,
    Tabbed,
    Tab,
    CruResource,
    Loading,
    Labels,
    ClusterMembershipEditor,
    Banner,
    LabeledInput,
    RadioGroup,
    KeyValue,
    ArrayList,
    ClusterAppearance,
    HostCluster
  },

  mixins: [CreateEditView],

  props: {
    mode: {
      type:     String,
      required: true,
    },

    value: {
      type:     Object,
      required: true,
    },

    provider: {
      type:     String,
      required: true,
    },
  },

  name: 'CruK3KCluster',

  async fetch() {
    const hash = {};

    if (this.$store.getters['management/schemaFor'](CAPI.RANCHER_CLUSTER)) {
      hash.provClusters = this.$store.dispatch('management/findAll', { type: CAPI.RANCHER_CLUSTER });
    }
    if (this.$store.getters['management/schemaFor'](MANAGEMENT.CLUSTER)) {
      hash.mgmtClusters = this.$store.dispatch('management/findAll', { type: MANAGEMENT.CLUSTER });
    }

    const res = await allHash(hash);

    this.provClusters = res.provClusters;

    if (this.mode === _CREATE) {
      this.k3kCluster = await this.$store.dispatch('management/create', cloneDeep(defaultCluster));
    } else {
      const ns = this.value.metadata.annotations['ui.rancher/k3k-namespace'] || '';
      const id = ns.split('k3k-')[0];
      const parentClusterId = this.value.metadata.annotations['ui.rancher/parent-cluster'] || '';

      const parentProvCluster = this.provClusters.find((c) => c?.mgmt?.id === parentClusterId);

      try {
        const res = await this.$store.dispatch('management/request', {
          url:    `/k8s/clusters/${ parentClusterId }/v1/k3k.io.clusters/${ ns }/${ id }`,
          method: 'GET',
        });

        this.k3kCluster = res.data[0] || {};
        this.parentCluster = parentProvCluster.id;
      } catch (e) {
        this.errors.push(e);
      }
    }

    this.k3sVersions = await this.$store.dispatch('management/request', { url: '/v1-k3s-release/releases' });
  },

  created() {
    this.registerAfterHook(this.saveRoleBindings, 'save-role-bindings');
  },

  watch: {
    k3sVersionOptions(neu = []) {
      if (this.mode === _CREATE && neu.length && !this.k3kCluster.spec.version) {
        this.k3kCluster.spec.version = neu[0];
      }
    }
  },

  data() {
    const t = this.$store.getters['i18n/t'];

    return {
      provClusters:  [],
      parentCluster: '',
      k3kCluster:    {},
      modeOptions:   [{ label: t('k3k.mode.shared'), value: 'shared' }, { label: t('k3k.mode.virtual'), value: 'virtual' }],
      k3sVersions:   []
    };
  },

  computed: {
    ...mapGetters({ t: 'i18n/withFallback' }),

    canManageMembers() {
      return canViewClusterMembershipEditor(this.$store);
    },

    k3sVersionOptions() {
      return (this.k3sVersions?.data || []).map((d) => d.version.replace('+', '-')).reverse();
    },

    localValue: {
      get() {
        return this.value;
      },
      set(newValue) {
        this.$emit('update:value', newValue);
      }
    }

  },

  methods: {
    onMembershipUpdate(update) {
      this['membershipUpdate'] = update;
    },

    async saveRoleBindings() {
      await this.value.waitForMgmt();

      if (this.membershipUpdate.save) {
        await this.membershipUpdate.save(this.value.mgmt.id);
      }
    },

    updateName({ name }) {
      this.k3kCluster.metadata.name = name;
      this.k3kCluster.metadata.namespace = `k3k-${ name }`;
    },

    async findNormanCluster() {
      const cluster = this.provClusters.find((c) => c.id === this.parentCluster);

      return await cluster.findNormanCluster();
    },

    // create the k3k cluster crd
    async createCluster() {
      const normanCluster = await this.findNormanCluster();

      const ns = {
        apiVersion: 'v1',
        kind:       'Namespace',
        metadata:
          { name: this.k3kCluster.metadata.namespace }
      };

      const nsyaml = saferDump(ns);

      await normanCluster.doAction('importYaml', { yaml: nsyaml });

      delete this.k3kCluster.type;
      const apply = saferDump(this.k3kCluster);

      await normanCluster.doAction('importYaml', { yaml: apply });

      return normanCluster.id;
    },

    // create import cluster command from new prov cluster
    // run a job to generate kubeconfig and run the import command on the virtual cluster
    async importCluster(clusterId) {
      const clusterToken = await this.value.getOrCreateToken();

      while (!clusterToken.command) {
        await new Promise((resolve) => setTimeout(resolve, 250));
      }

      const command = clusterToken.command.split(' ');
      const registrationUrl = command[command.length - 1];

      let importJob = JSON.stringify(importJobTemplate).replaceAll(/K3K_NAMESPACE/g, this.value.metadata.name);

      importJob = importJob.replaceAll(/__url/g, registrationUrl);

      const importConfigMap = JSON.stringify(importConfigMapTemplate).replaceAll(/K3K_NAMESPACE/g, this.value.metadata.name);

      const importJobYaml = saferDump(JSON.parse(importJob));

      const configMapYaml = saferDump(JSON.parse(importConfigMap));

      const applyCm = {
        defaultNamespace: this.value.metadata.name,
        yaml:             configMapYaml
      };

      const applyJob = {
        defaultNamespace: this.value.metadata.name,
        yaml:             importJobYaml
      };

      await this.$store.dispatch('management/request', {
        url:    `/v1/management.cattle.io.clusters/${ clusterId }?action=apply`,
        method: 'POST',
        data:   applyCm
      });

      await this.$store.dispatch('management/request', {
        url:    `/v1/management.cattle.io.clusters/${ clusterId }?action=apply`,
        method: 'POST',
        data:   applyJob
      });
    },

    async saveOverride(btnCb) {
      try {
        if (this.mode === _CREATE) {
          // create the k3k cluster crd and return the id of the host cluster's mgmt cluster
          // mgmt cluster is needed to make requests against its steve api
          // prov cluster is needed to generate a human-readable host cluster name
          const clusterId = await this.createCluster();
          const parentProvCluster = this.provClusters.find((c) => c.id === this.parentCluster);

          // Add annotations so the ui knows the imported cluster is a virtual cluster, and which is its parent cluster
          this.value.metadata = this.value.metadata || {};
          this.value.metadata.annotations = { ...defaultAnnotations };

          this.value.metadata.annotations['ui.rancher/parent-cluster'] = clusterId;

          this.value.metadata.annotations['ui.rancher/parent-cluster-display'] = parentProvCluster.displayName || parentProvCluster.name;
          this.value.metadata.annotations['ui.rancher/k3k-namespace'] = `k3k-${ this.value.metadata.name }`;

          // get import cluster command
          this.importCluster(clusterId);
        } else {
          // save existing k3kCluster
          const cluster = await this.findNormanCluster();

          await cluster.$dispatch('request', {
            url:    `/k8s/clusters/${ cluster.id }/v1/k3k.io.clusters/${ this.value.metadata.namescape }/${ this.value.metadata.name }`,
            method: 'PUT',
            data:   this.k3kCluster
          });
        }
        // save prov cluster
        await this.save(btnCb);
      } catch (err) {
        this.errors.push(err);
        btnCb(false);
      }
    },

    cancel() {
      this.$router.push({
        name:   'c-cluster-product-resource',
        params: {
          cluster:  this.$route.params.cluster,
          product:  this.$store.getters['productId'],
          resource: CAPI.RANCHER_CLUSTER,
        },
      });
    },
  }
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <CruResource
    v-else
    :mode="mode"
    :resource="value"
    :errors="errors"
    component-testid="cluster-manager-virtual-cluster"
    :cancel-event="true"
    @finish="saveOverride"
    @error="e => errors = e"
    @cancel="cancel"
  >
    <NameNsDescription
      v-if="!isView"
      v-model:value="localValue"
      :mode="mode"
      :namespaced="false"
      name-label="cluster.name.label"
      name-placeholder="cluster.name.placeholder"
      description-label="cluster.description.label"
      description-placeholder="cluster.description.placeholder"
      :create-namespace-override="true"
      @update:value="updateName"
    >
      <template #customize>
        <ClusterAppearance
          :name="k3kCluster.metadata.name"
          :current-cluster="value"
          :mode="mode"
        />
      </template>
    </NameNsDescription>

    <Tabbed
      :side-tabs="true"
      default-tab="virtual-cluster"
    >
      <Tab
        name="virtual-cluster"
        label-key="k3k.tabs.basics"
        :weight="5"
      >
        <HostCluster
          v-model:parent-cluster="parentCluster"
          :mode="mode"
          :clusters="provClusters"
        />

        <div class="row mb-20">
          <div class="col span-6">
            <LabeledSelect
              v-model:value="k3kCluster.spec.version"
              label-key="k3k.k3sVersion.label"
              :options="k3sVersionOptions"
              :mode="mode"
            />
          </div>
        </div>

        <div class="row mb-20">
          <div class="col span-6">
            <RadioGroup
              v-model:value="k3kCluster.spec.mode"
              name="k3k-cluster-mode"
              :row="true"
              :mode="mode"
              label-key="k3k.mode.label"
              :options="[{label: t('k3k.mode.shared'), value: 'shared'},{label: t('k3k.mode.virtual'), value: 'virtual'} ]"
            >
              <template #label>
                <h4>{{ t('k3k.mode.label') }}</h4>
              </template>
            </RadioGroup>
          </div>
          <div class="col span-6">
            <span class="text-muted">{{ t('k3k.mode.tooltip') }}</span>
          </div>
        </div>

        <h4>{{ t('k3k.servers.title') }}</h4>
        <div class="row mb-20">
          <div class="col span-3">
            <LabeledInput
              v-model:value.number="k3kCluster.spec.servers"
              label-key="k3k.servers.label"
              :mode="mode"
            />
          </div>
          <div class="col span-3">
            <LabeledInput
              v-model:value.number="k3kCluster.spec.agents"
              label-key="k3k.agents.label"
              :mode="mode"
            />
          </div>
        </div>

        <div class="row mb-20">
          <div class="col span-12">
            <KeyValue
              v-model:value="k3kCluster.spec.nodeSelector"
              :initial-empty-row="true"
              :mode="mode"
              :read-allowed="false"
              :title="t('k3k.nodeSelector.label')"
              :add-label="t('k3k.nodeSelector.addLabel')"
            >
              <template #title>
                <h4>{{ t('k3k.nodeSelector.label') }}</h4>
                <span class="text-muted">{{ t('k3k.nodeSelector.tooltip') }}</span>
              </template>
            </KeyValue>
          </div>
        </div>
      </Tab>
      <Tab
        name="networking"
        label-key="k3k.tabs.networking"
        :weight="4"
      >
        <div class="row mb-20">
          <div class="col span-6">
            <LabeledInput
              v-model:value="k3kCluster.spec.clusterCIDR"
              label-key="k3k.clusterCIDR.label"
              placeholder-key="k3k.clusterCIDR.placeholder"
              :mode="mode"
            />
          </div>
        </div>
        <div class="row mb-20">
          <div class="col span-6">
            <LabeledInput
              v-model:value="k3kCluster.spec.serviceCIDR"
              label-key="k3k.serviceCIDR.label"
              placeholder-key="k3k.serviceCIDR.placeholder"
              :mode="mode"
            />
          </div>
          <div class="col span-6 centered">
            <t
              k="k3k.serviceCIDR.tooltip"
              class="text-label"
            />
          </div>
        </div>
        <div class="row mb-20">
          <div class="col span-6">
            <LabeledInput
              v-model:value="k3kCluster.spec.clusterDNS"
              label-key="k3k.clusterDNS.label"
              placeholder-key="k3k.clusterDNS.placeholder"
              :mode="mode"
            />
          </div>
        </div>
        <div class="row mb-20">
          <div class="col span-6">
            <ArrayList
              v-model:value="k3kCluster.spec.tlsSANs"
              :protip="false"
              :mode="mode"
              :title="t('k3k.tlsSANs.label')"
            />
          </div>
        </div>
      </Tab>
      <Tab
        v-if="canManageMembers"
        name="memberRoles"
        label-key="cluster.tabs.memberRoles"
        :weight="3"
      >
        <Banner
          v-if="isEdit"
          color="info"
        >
          {{ t('cluster.memberRoles.removeMessage') }}
        </Banner>
        <ClusterMembershipEditor
          :mode="mode"
          :parent-id="value.mgmt ? value.mgmt.id : null"
          @membership-update="onMembershipUpdate"
        />
      </Tab>
      <Labels
        v-model:value="localValue"
        :mode="mode"
      />
    </Tabbed>
  </CruResource>
</template>

<style lang="scss" scoped>
  .centered {
    display: flex;
    align-items: center;
  }
</style>
