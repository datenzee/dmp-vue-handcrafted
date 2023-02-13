<script setup>
import * as $rdf from 'rdflib'
import { reactive } from "@vue/reactivity";
import axios from 'axios'
import moment from 'moment'

const state = reactive({
  object: null,
  graph: null,
  loading: true,
  error: null,
})

const url = new URL(window.location.href)
const root = url.searchParams.get('root') || 'https://datenzee.ds-wizard.org/projects/9a710add-11b7-4544-a432-82727b45eca6#DMP_4f48219d-17e2-4d71-8fd6-6c42ad3af7bf'
state.object = $rdf.sym(root)

const dataUrl = url.searchParams.get('data')
if (dataUrl) {
  axios
    .get(dataUrl)
    .then((result) => {
      try {
        state.graph = new $rdf.IndexedFormula()
        $rdf.parse(result.data, state.graph, state.object.uri, 'text/turtle')
        state.loading = false
      } catch (error) {
        state.loading = false
        state.error = error.toString()
      }
    })
}

const formatDate = (date) => {
  return moment(date).format('MMMM Do YYYY')
}

const getValue = (object, predicate) => {
  return state.graph.match(object.object, $rdf.sym(predicate))[0]?.object.value
}
const getValueDate = (object, predicate) => {
  return formatDate(getValue(object, predicate))
}

const getContacts = () => {
  return state.graph.match(state.object, $rdf.sym('https://w3id.org/dcso/ns/core#hasContact')) || []
}

const getContributors = () => {
  return state.graph.match(state.object, $rdf.sym('https://w3id.org/dcso/ns/core#hasContributor')) || []
}

const getProjects = () => {
  return state.graph.match(state.object, $rdf.sym('https://w3id.org/dcso/ns/core#hasProject')) || []
}

const getDatasets = () => {
  return state.graph.match(state.object, $rdf.sym('https://w3id.org/dcso/ns/core#hasDataset')) || []
}

const getDistributions = (dataset) => {
  return state.graph.match(dataset.object, $rdf.sym('https://w3id.org/dcso/ns/core#hasDistribution')) || []
}

const getLicenseValue = (distribution, value) => {
  const license = getValue(distribution, 'https://w3id.org/dcso/ns/core#hasLicense')

  if (license) {
    return getValue(license, value)
  }

  return ''
}
const getLicenseValueDate = (distribution, value) => {
  const license = getValue(distribution, 'https://w3id.org/dcso/ns/core#hasLicense')

  if (license) {
    return getValueDate(license, value)
  }

  return ''
}
</script>
<template>
  <main>
    <div v-if="state.loading">Loading...</div>
    <div v-else-if="state.error">{{ state.error }}</div>
    <div v-else>
      <h1>{{ getValue(state, 'http://purl.org/dc/terms/title') }}</h1>
      <p>
        <strong>Data Management Plan</strong>
        <br>
        {{ getValue(state, 'http://purl.org/dc/terms/description') }}
      </p>

      <h2>Contacts</h2>
      <p>There are the following contacts related to the project of this DMP:</p>
      <ul>
        <li v-for="(contact, index) in getContacts()" :key="index">
          <strong>{{ getValue(contact, 'http://xmlns.com/foaf/0.1/name') }}</strong>
          (<a :href="`mailto: ${getValue(contact, 'http://xmlns.com/foaf/0.1/mbox')}`">{{
            getValue(contact, 'http://xmlns.com/foaf/0.1/mbox')
          }}</a>)
        </li>
      </ul>

      <h2>Contributor</h2>
      <p>There are the following contributors related to the project of this DMP:</p>
      <ul>
        <li v-for="(contributor, index) in getContributors()" :key="index">
          <strong>{{ getValue(contributor, 'http://xmlns.com/foaf/0.1/name') }}</strong>
          (<a :href="`mailto: ${getValue(contributor, 'http://xmlns.com/foaf/0.1/mbox')}`">{{
            getValue(contributor.object, 'http://xmlns.com/foaf/0.1/mbox')
          }}</a>), Role: <em>{{ getValue(contributor, 'https://w3id.org/dcso/ns/core#role') }}</em>
        </li>
      </ul>

      <h2>Projects</h2>
      <p>We will be working on the following project and for those are the data and work described in this DMP.</p>
      <div v-for="(project, index) in getProjects()" :key="index">
        <h3>{{ getValue(project, 'http://purl.org/dc/terms/title') }}</h3>
        <p>{{ getValue(project, 'http://purl.org/dc/terms/description') }}</p>
        <div>
          <strong>Start date:</strong> {{ getValueDate(project, 'https://w3id.org/dcso/ns/core#start') }}<br>
          <strong>End date:</strong> {{ getValueDate(project, 'https://w3id.org/dcso/ns/core#end') }}<br>
        </div>
      </div>

      <h2>Datasets</h2>
      <p>We will reuse or produce the following datasets in the project.</p>

      <div v-for="(dataset, index) in getDatasets()" :key="index">
        <h3>{{ getValue(dataset, 'http://purl.org/dc/terms/title') }}</h3>
        <p>{{ getValue(dataset, 'http://purl.org/dc/terms/description') }}</p>
        <div>
          <strong>Keywords:</strong> {{ getValue(dataset, 'http://www.w3.org/ns/dcat#keyword') }}<br>
          <strong>Issued:</strong> {{ getValueDate(dataset, 'http://purl.org/dc/terms/issued') }}<br>
          <strong>Dataset Type:</strong> {{ getValue(dataset, 'https://w3id.org/dcso/ns/core#datasetType') }}<br>
        </div>

        <p v-if="getValue(dataset, 'https://w3id.org/dcso/ns/core#personalData') == 'yes'">The dataset contains personal
          data.</p>
        <p v-else-if="getValue(dataset, 'https://w3id.org/dcso/ns/core#personalData') == 'no'">The dataset contains no
          personal data.</p>
        <p v-else>It is unclear if the dataset contains personal data.</p>

        <p v-if="getValue(dataset, 'https://w3id.org/dcso/ns/core#sensitiveData') == 'yes'">The dataset contains sensitive data.</p>
        <p v-else-if="getValue(dataset, 'https://w3id.org/dcso/ns/core#sensitiveData') == 'no'">The dataset contains no sensitive data.</p>
        <p v-else>It is unknown if the dataset contains sensitive data.</p>

        <p>{{ getValue(dataset, 'https://w3id.org/dcso/ns/core#preservationStatement') }}</p>

        <h4>Distributions</h4>
        <p>This dataset will be available in the following distributions.</p>
        <div v-for="(distribution, index) in getDistributions(dataset)" :key="index">
          <h5>{{ getValue(distribution, 'http://purl.org/dc/terms/title') }}</h5>
          <p>{{ getValue(distribution, 'http://purl.org/dc/terms/description') }}</p>

          <div>
            <strong>Format: </strong> {{ getValue(distribution, 'http://purl.org/dc/terms/format') }}<br>
            <strong>Byte Size: </strong> {{ getValue(distribution, 'http://www.w3.org/ns/dcat#byteSize') }}<br>
            <strong>Access URL: </strong> <a :href="getValue(distribution, 'http://www.w3.org/ns/dcat#accessURL')">{{ getValue(distribution, 'http://www.w3.org/ns/dcat#accessURL') }}</a><br>
            <strong>Download URL: </strong> <a :href="getValue(distribution, 'http://www.w3.org/ns/dcat#downloadURL')">{{ getValue(distribution, 'http://www.w3.org/ns/dcat#downloadURL') }}</a><br>
            <strong>Availble Until: </strong> {{ getValueDate(distribution, 'https://w3id.org/dcso/ns/core#availableUntil') }}<br>
            <strong>Data Access: </strong> {{ getValue(distribution, 'https://w3id.org/dcso/ns/core#dataAccess') }}
          </div>

          <p>From {{ getLicenseValueDate(distribution, 'https://w3id.org/dcso/ns/core#startDate') }}, the distribution uses the following license: <a :href="getLicenseValue(distribution, 'https://w3id.org/dcso/ns/core#licenseRef')">{{ getLicenseValue(distribution, 'https://w3id.org/dcso/ns/core#licenseRef') }}</a>.</p>
        </div>
      </div>
    </div>
  </main>
</template>
