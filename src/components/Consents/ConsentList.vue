<template>
  <div>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
      <a class="navbar-brand" href="#"> {{ title }} </a>

      <el-button type="primary" icon="el-icon-refresh"
        @click="$emit('consents-refresh')"></el-button>
    </nav>

    <el-collapse v-model="expanded_items">
      <ul class="list">
        <el-collapse-item
          v-for="(consent, index) in consents"
          :title="consent.label"
          :name="consent.label + index"
          :key="index">
          <el-row>
            <vue-json-pretty :deep=0 :data="consent" />
            <el-button size="medium"
              @click="preview(consent)">Preview</el-button>
          </el-row>
        </el-collapse-item>
      </ul>
    </el-collapse>
  </div>
</template>

<script>
import axios from 'axios';

import VueJsonPretty from 'vue-json-pretty';
import { renderForm } from 'odca-form'

export default {
  name: 'consent-list',
  props: [
    'title',
    'consents'
  ],
  components: {
    VueJsonPretty
  },
  data () {
    return {
      expanded_items:[],
    }
  },
  computed: {
    ocaRepoUrl: function() {
      return `${config.env.VUE_APP_PROTOCOL}://${config.env.VUE_APP_OCA_REPO}.${config.env.VUE_APP_HOST}`
    },
    localDataVaultUrl: function() {
      return this.$session.get('localDataVaultUrl')
    },
  },
  methods: {
    async renderConsentForm(consent) {
      const consentAnswers = (await axios.get(`${this.localDataVaultUrl}/api/v1/files/${consent.dataDri}`)).data
      const consentBranch = (await axios.get(
        `${this.ocaRepoUrl}/api/v2/schemas/${consent.ocaSchemaNamespace}/${consent.ocaSchemaDri}`
      )).data

      const consentLangBranches = this.splitBranchPerLang(consentBranch)
      let consentForm
      const consentFormAlternatives = []
      try {
        consentLangBranches.forEach(langBranch => {
          consentFormAlternatives.push({
            language: langBranch.lang,
            form: renderForm([langBranch.branch.schema_base, ...langBranch.branch.overlays]).form
          })
        })
        consentForm = consentFormAlternatives[0].form
      } catch(e) {
        console.log(e)
        this.$noty.error("ERROR! Consent form data are corrupted.", {
          timeout: 1000
        })
      }

      return {
        form: consentForm,
        formAlternatives: consentFormAlternatives,
        answers: consentAnswers
      }
    },
    splitBranchPerLang(branch) {
      const langBranches = []
      const labelOverlays = branch.overlays.filter(o => o.type.includes("label"))
      const languages = labelOverlays.map(o => o.language)
      const schemaBase = branch.schema_base
      languages.forEach(lang => {
        langBranches.push({
          lang: lang,
          branch: {
            schema_base: schemaBase,
            overlays: branch.overlays.filter(o => {
              if(!o.language) { return true }
              return o.language == lang
            })
          }
        })
      })
      return langBranches
    },
    async preview(consent) {
      this.$emit('consent-preview', await this.renderConsentForm(consent))
    }
  },
}
</script>