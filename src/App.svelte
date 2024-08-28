<script lang="ts">
  import { onMount } from 'svelte';
  import {CLINET_ID, CODE_VERIFIER_LOCAL_STORAGE_KEY, FHIR_BASE_URL, REDIRECT_URL as REDIRECT_URI, SMART_AUTH_URL, SMART_TOKEN_URL, TOKEN_RESPONSE_LOCAL_STORAGE_KEY} from './config'
  import axios from 'axios';
  import pkceChallenge from "pkce-challenge";

  import { Button, ButtonGroup, Icon, Navbar, Spinner, TabContent, TabPane } from '@sveltestrap/sveltestrap';
  import { colorMode, useColorMode } from '@sveltestrap/sveltestrap';

  import "carbon-components-svelte/css/all.css";

  import PatientDetails from './lib/PatientDetails.svelte';
  import MedicationDetails from './lib/MedicationDetails.svelte';
  import ObservationViewer from './lib/ObservationViewer.svelte';
  import VitalSigns from './lib/VitalSigns.svelte';

  // import {
  //   Theme,
  //   RadioButtonGroup,
  //   RadioButton,
  // } from "carbon-components-svelte";

  // let theme:any = "g90";

  let tokenResponse: {
    access_token: string,
    id_token: string,
    patient: string,
    scope: string,
  }
  let loading = true

  const getSecs = (date: Date) => {
    return Math.round((date).getTime() / 1000)
  }

  const generateCodeChallenge = async () => {
    const {code_challenge, code_verifier} = await pkceChallenge()
    localStorage.setItem(CODE_VERIFIER_LOCAL_STORAGE_KEY, code_verifier)
    return code_challenge
  }

  const generateRedirectUrl = (codeChallenge: string) =>{
    const authorizationUrl = new URL(SMART_AUTH_URL)
    authorizationUrl.searchParams.set('client_id', CLINET_ID)
    authorizationUrl.searchParams.set('scope', 'openid fhirUser offline')
    authorizationUrl.searchParams.set('redirect_uri', REDIRECT_URI)
    authorizationUrl.searchParams.set('response_type', 'code')
    // authorizationUrl.searchParams.set('state', '1234567')
    authorizationUrl.searchParams.set('aud', FHIR_BASE_URL)
    authorizationUrl.searchParams.set('code_challenge', codeChallenge)
    authorizationUrl.searchParams.set('code_challenge_method', 'S256')
    return authorizationUrl.href
  }

  const makeTokenRequest = async (code: string, codeVerifier: string) => {
    const tokenRequestForm = new FormData();
    tokenRequestForm.set('grant_type', 'authorization_code')
    tokenRequestForm.set('code', code)
    tokenRequestForm.set('redirect_uri', REDIRECT_URI)
    tokenRequestForm.set('client_id', CLINET_ID)
    tokenRequestForm.set('code_verifier', codeVerifier)
    const tokenGeneratedAt = getSecs(new Date())
    const response = await axios.postForm(SMART_TOKEN_URL, tokenRequestForm)
    tokenResponse = response.data
    localStorage.setItem(TOKEN_RESPONSE_LOCAL_STORAGE_KEY, JSON.stringify({...tokenResponse, issued_at_in_secs: tokenGeneratedAt}))
  }

  const logout = () => {
    localStorage.removeItem(TOKEN_RESPONSE_LOCAL_STORAGE_KEY)
    tokenResponse.access_token = '';
  }

  onMount(async ()=>{
    const code = new URL(window.location.href).searchParams.get('code')
    const codeVerifier = localStorage.getItem(CODE_VERIFIER_LOCAL_STORAGE_KEY)
    const tokenResponseString = localStorage.getItem(TOKEN_RESPONSE_LOCAL_STORAGE_KEY);
    if (tokenResponseString) {
      const tokenResponseTemp = JSON.parse(tokenResponseString)
      const {issued_at_in_secs} = tokenResponseTemp
      const expires_in_secs = issued_at_in_secs + tokenResponseTemp.expires_in
      const now_in_secs = getSecs(new Date())
      if (now_in_secs >= expires_in_secs) {
        localStorage.removeItem(TOKEN_RESPONSE_LOCAL_STORAGE_KEY)
      } else {
        tokenResponse = tokenResponseTemp
      }
    }
    if (!tokenResponse) {
      if (code && codeVerifier) {
        await makeTokenRequest(code, codeVerifier)
        localStorage.removeItem(CODE_VERIFIER_LOCAL_STORAGE_KEY)
      }
    }
    loading = false
  })

  export const initiateAuthorizationRequest = async ()=>{
    const codeChallenge = await generateCodeChallenge()
    const redirectUrl = generateRedirectUrl(codeChallenge)
    window.location.href = redirectUrl
  }

</script>

<svelte:head>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.2/font/bootstrap-icons.css">
</svelte:head>

<main>
  <Navbar container="xl" expand="md" sticky="" color="primary-subtle">
    <h1>Goodlicious Health Care</h1>
    <ButtonGroup style="float:center">
      <Button color="primary" outline active={$colorMode === 'light'} on:click={() => useColorMode('light')}>
        light <Icon name="sun-fill" />
      </Button>
      <Button color="primary" outline active={$colorMode === 'dark'} on:click={() => useColorMode('dark')}>
        dark <Icon name="moon-stars-fill" />
      </Button>
      <Button color="primary" outline active={$colorMode === 'auto'} on:click={() => useColorMode('auto')}>
        auto <Icon name="circle-half" />
      </Button>
    </ButtonGroup>
    {#if tokenResponse && tokenResponse.access_token != ''}
      <div style="float: right;">
        <Button class="logout-btn" color="danger" on:click={logout}>Logout</Button>
      </div>
    {/if}
  </Navbar>
  {#if loading}
    <center>
      <Spinner style="margin-top: 4em"/>
    </center>
  {:else}
    {#if tokenResponse && tokenResponse.access_token != ''}
      <div style="margin: 50px">
        <PatientDetails accessToken={tokenResponse.access_token} patientId={tokenResponse.patient} />

        <TabContent style="padding-top: 10px">
          <TabPane tabId="tab1" active>
            <span slot="tab">
              <Icon name="capsule" /> Medications
            </span>
            <MedicationDetails accessToken={tokenResponse.access_token} patientId={tokenResponse.patient}></MedicationDetails>
          </TabPane>
          <TabPane tabId="tab2">
            <span slot="tab">
              <Icon name="journal-medical" /> Lab Tests
            </span>
            <ObservationViewer accessToken={tokenResponse.access_token} patientId={tokenResponse.patient}></ObservationViewer>
          </TabPane>
          <TabPane tabId="tab3">
            <span slot="tab">
              <Icon name="clipboard2-pulse" /> Vital Signs
            </span>
            <VitalSigns accessToken={tokenResponse.access_token} patientId={tokenResponse.patient}></VitalSigns>
          </TabPane>
        </TabContent>
      </div>
    {:else}
      <div class="loginBox" style="margin-top: 4em">
        <center>
          <Button class="login-btn" color="primary" on:click={initiateAuthorizationRequest}>Sign in with Epic</Button>
        </center>
      </div>
    {/if}
  {/if}
</main>
