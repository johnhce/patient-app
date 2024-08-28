<script lang="ts">
  import axios from "axios";
  import { FHIR_BASE_URL } from "../config";
  import {formatRelative, subDays} from 'date-fns'
  import type { Bundle, BundleEntry, MedicationRequest, Observation, OperationOutcome } from "fhir/r4";

  import { Column, Spinner, Table } from "@sveltestrap/sveltestrap"

  export let accessToken: string
  export let patientId: string
  export let category: string = 'laboratory'

  const getLabResults = async () => {
    const labObservationResponse = await axios.get<Bundle<Observation | OperationOutcome>>(`${FHIR_BASE_URL}/Observation`, {
      params: {
        subject: patientId,
        category,
//        sort: 'date'
      },
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    })
    return labObservationResponse.data
  }

  const getObservationDisplay = (observation: Observation | undefined) => {
    if (!observation) {
      return ''
    }
    const isBp = observation.code?.coding?.find(a=>a.code === '55284-4')
    if (isBp) {
      const systolicComponent = observation.component?.find(a=>a.code?.coding?.find(b=>b.code==='8480-6'))
      const systolic = systolicComponent?.valueQuantity?.value
      const diastolicComponent = observation.component?.find(a=>a.code?.coding?.find(b=>b.code==='8462-4'))
      const diastolic = diastolicComponent?.valueQuantity?.value
      return `${systolic}/${diastolic}`
    }
    if (!observation?.valueQuantity?.unit) {
      return observation?.valueQuantity?.value
    }
    return `${observation?.valueQuantity?.value} ${observation?.valueQuantity?.unit}`

  }

  const getObservationEntries = (bundle: Bundle<Observation | OperationOutcome>): BundleEntry<Observation>[] => {
    if (!bundle?.entry) {
      return []
    }
    const results = bundle.entry?.filter((entry) => entry.resource?.resourceType === 'Observation') as BundleEntry<Observation>[];
    return results.sort((a,b)=>{
      if (a?.resource?.effectiveDateTime && b?.resource?.effectiveDateTime) {
        return new Date(b?.resource?.effectiveDateTime).getTime() - new Date(a?.resource?.effectiveDateTime).getTime();
      }
      return 0
    })
    
  }
</script>
<div class="mt-10 max-w-md mx-auto">
{#await getLabResults()}
  <center>
    <Spinner style="margin-top: 4em"/>
  </center>
{:then labResults} 
  {#if getObservationEntries(labResults).length == 0}
    <div style="margin-top: 1em; margin-left: 4em">
      No lab tests for this patient.
    </div>
  {:else}
    <Table striped style="margin-top: 1em;">
      <thead>
        <tr>
          <th>Name</th>
          <th>Date</th>
          <th>Measurement</th>
        </tr>
      </thead>
      <tbody>
      {#each getObservationEntries(labResults) as observation, i}
      <tr>
        <td>{observation.resource?.code?.text}</td>
        <td>{#if observation?.resource?.effectiveDateTime}
          {formatRelative(new Date(observation?.resource.effectiveDateTime), new Date())}
        {/if}</td>
        <td>{getObservationDisplay(observation.resource)}</td>
      </tr>
      {/each}
    </tbody>
  </Table>
  {/if}
{/await}
</div>