<script lang="ts">
  import axios from "axios";
  import { FHIR_BASE_URL } from "../config";
  import { add, formatRelative, subDays } from 'date-fns'
  import type { Bundle, BundleEntry, MedicationRequest, Observation, OperationOutcome } from "fhir/r4";

  import { Column, Spinner, Table } from "@sveltestrap/sveltestrap";
  import { DataTable, Pagination } from "carbon-components-svelte";
  import { te } from "date-fns/locale";

  export let accessToken: string
  export let patientId: string

  // for datatable
  interface TableEntry {
    id: number;
    name?: string;
    date?: string;
    val?: string;
  }
  let pageSize = 10;
  let page = 1;
  let rows:Array<TableEntry> = [];
  // Array.from({ length: 25 }).map((_, i) => ({
  //   id: i,
  //   name: "Load Balancer " + (i + 1),
  //   date: "HTTP",
  //   val: 3000 + i * 10,
  // }));

  const getLabResults = async () => {
    const labObservationResponse = await axios.get<Bundle<Observation | OperationOutcome>>(`${FHIR_BASE_URL}/Observation`, {
      params: {
        subject: patientId,
        category: 'vital-signs',
        _sort: '-_lastUpdated',
        // _count: 20,
        // _skip: (page-1)*20,
        // _total: 'accurate'
      },
      headers: {
        'Authorization': `Bearer ${accessToken}`,
//        'Prefer': 'pagination=offset-skip',
      }
    })
//    initTableData(labObservationResponse.data);
  // rows = Array.from({ length: 10 }).map((_, i) => ({
  //   id: i,
  //   name: labObservationResponse.data,//:String,
  //   date: "HTTP",
  //   val: 3000 + i * 10,
  // }));
  let entries = getObservationEntries(labObservationResponse.data);
  for(let i = 0; i < entries.length; i++) {
    let te = {} as TableEntry;
    te.id = i;
    te.name = entries[i].resource?.code?.text;
    let dt = entries[i]?.resource?.effectiveDateTime;
    if (dt != undefined)
      te.date = formatRelative(new Date(dt), new Date());
    else
      te.date = '';
    te.val = getObservationDisplay(entries[i].resource)?.toString();
    rows.push(te);
//     rows.push({
//       name: entries[i].resource?.code?.text,
// //      date: formatRelative(new Date(entries[i]?.resource.effectiveDateTime), new Date()),
//       val: getObservationDisplay(entries[i].resource),
//     });
  }

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

  const initTableData = (labResults:Bundle<Observation | OperationOutcome>) => {
    let n = 0;
//    rows[n].id = n;
    // try { rows[n].name = JSON.stringify(labResults.entry); } catch(ex) {alert(ex);}
    // try { rows[n].date = JSON.stringify(labResults.entry?.entries); } catch(ex) {alert(ex);}
    // try { rows[n].val = JSON.stringify(labResults.entry?.entries.toString); } catch(ex) {alert(ex);}
    n++;
    // (Object.keys(labResults) as (keyof typeof labResults)[]).forEach((key, index) => {
    //   let observation:BundleEntry<Observation | OperationOutcome> = labResults[key];
    //   rows.add({
    //      name: observation.resource?.code?.text,
    //      date: formatRelative(new Date(observation?.resource.effectiveDateTime), new Date()),
    //      val: getObservationDisplay(observation.resource),
    //   });
    // });
    //forEach ((observation:Observation | undefined) => {
      // observationEntries:BundleEntry<Observation> = getObservationEntries(labResults);
      // rows.add({
      //   name: observation.resource?.code?.text,
      //   date: formatRelative(new Date(observation?.resource.effectiveDateTime), new Date()),
      //   val: getObservationDisplay(observation.resource),
      // });
    //});
  }
</script>
<div class="mt-10 max-w-md mx-auto">
{#await getLabResults()}
  <center>
    <Spinner style="margin-top: 4em"/>
  </center>
{:then labResults} 
  <DataTable
    sortable
    title="Vital Signs"
    description="Your key health measurements."
    headers={[
      { key: "name", value: "Name" },
      { key: "date", value: "Date" },
      { key: "val", value: "Measurement" },
    ]}
    {pageSize}
    {page}
    {rows}
  />
  <Pagination
    bind:pageSize
    bind:page
    totalItems={rows.length}
    pageSizeInputDisabled
  />

  <!--
  <Table rows={rows} let:row striped>
    <Column header="Name">
      {row.name}
    </Column>
    <Column header="Date">
      {row.date}
    </Column>
    <Column header="Measurement">{row.val}</Column>
  </Table>-->

  <!-- {#each getObservationEntries(labResults) as observation, i}
  <p>
    <span class="font-medium">
      {observation.resource?.code?.text}
      {#if observation?.resource?.effectiveDateTime}
        ({formatRelative(new Date(observation?.resource.effectiveDateTime), new Date())})
      {/if}
      :
    </span>
    {getObservationDisplay(observation.resource)}
  </p>
  <div class="ml-4">
  </div>
  {/each} -->
{/await}
</div>