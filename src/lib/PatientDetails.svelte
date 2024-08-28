<script lang="ts">
  import axios from "axios";
  import { FHIR_BASE_URL } from "../config";
  import type { Patient } from "fhir/r4";
  import { Spinner } from "@sveltestrap/sveltestrap"

  export let accessToken: string
  export let patientId: string

  const getPatientDetails = async () => {
    const patientResponse = await axios.get<Patient>(`${FHIR_BASE_URL}/Patient/${patientId}`, {
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    })
    return patientResponse.data
  }
</script>
<div class="mt-10 max-w-md mx-auto">
{#await getPatientDetails()}
  <center>
    <Spinner style="margin-top: 4em"/>
  </center>
{:then patient} 
  <h2 class="text-xl">
    Hello {patient?.name?.[0]?.given?.[0]},
  </h2>
  <h3>Welcome to your patient record!</h3>
  <p><b>Full Name:</b> {patient?.name?.[0]?.text}</p>
  <p><b>Epic identifier:</b> {patient?.identifier?.find(i=>i.system==='urn:oid:1.2.840.114350.1.13.0.1.7.5.737384.0')?.value}</p>
  <p><b>Date of birth:</b> {patient?.birthDate}</p>
  <p><b>Gender:</b> <span class="capitalize">{patient?.gender}</span></p>
{/await}
</div>