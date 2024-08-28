<script lang="ts">
  import axios from "axios";
  import { FHIR_BASE_URL } from "../config";
  import type { Bundle, BundleEntry, MedicationRequest, OperationOutcome } from "fhir/r4";
  import { Accordion, AccordionItem, Spinner } from "@sveltestrap/sveltestrap";

  export let accessToken: string
  export let patientId: string

  const getMedications = async () => {
    const medicationRequestResponse = await axios.get<Bundle<MedicationRequest | OperationOutcome>>(`${FHIR_BASE_URL}/MedicationRequest`, {
      params: {
        subject: patientId
      },
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    })
    return medicationRequestResponse.data
  }

  const getMedicationRequestEntries = (bundle: Bundle<MedicationRequest | OperationOutcome>): BundleEntry<MedicationRequest>[] => {
    if (!bundle?.entry) {
      return []
    }
    return bundle.entry?.filter((entry) => entry.resource?.resourceType === 'MedicationRequest') as BundleEntry<MedicationRequest>[]
  }
</script>
<div class="mt-10 max-w-md mx-auto">
{#await getMedications()}
  <center>
    <Spinner style="margin-top: 4em"/>
  </center>
{:then medicationList} 
  {#if getMedicationRequestEntries(medicationList).length == 0}
    There are no medications on your chart.
  {:else}
    <Accordion>
    {#each getMedicationRequestEntries(medicationList) as medication, i}
      <AccordionItem header="{i + 1}. {medication.resource?.medicationReference?.display}">
        <div style="margin-left: 4em">
          {#if medication?.resource?.dosageInstruction?.[0]?.patientInstruction}
          <p>
            Dosage: {medication?.resource?.dosageInstruction?.[0]?.patientInstruction}
          </p>
          {/if}
          {#if medication?.resource?.reasonCode?.[0]?.text}
          <p>
            Reason: {medication?.resource?.reasonCode?.[0]?.text}
          </p>
          {/if}
        </div>
      </AccordionItem>
    {/each}
    </Accordion>
{/if}
{/await}
</div>