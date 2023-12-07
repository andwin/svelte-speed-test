<script lang="ts">
  let baseUrl: string = 'https://www.example.com'
  let urlsText: string = '/'

  let requestsToSend: string = '10'
  const simultaneousRequests = 4

  const loadDataFromUrl = () => {
    const url = new URL(window.location.toString())
    const params = new URLSearchParams(url.search)
    const baseUrlParam = params.get('baseUrl')
    const urlsParam = params.get('urls')
    const requestsToSendParam = params.get('requestsToSend')

    if (baseUrlParam) {
      baseUrl = baseUrlParam
    }
    if (urlsParam) {
      urlsText = urlsParam
    }
    if (requestsToSendParam) {
      requestsToSend = requestsToSendParam
    }
  }
  loadDataFromUrl()

  const saveDataToUrl = () => {
    const url = new URL(window.location.toString())
    const params = new URLSearchParams(url.search)
    params.set('baseUrl', baseUrl)
    params.set('urls', urlsText)
    params.set('requestsToSend', requestsToSend)
    window.history.replaceState(null, '', window.location.pathname + '?' + params.toString())
  }
  $: baseUrl, urlsText, requestsToSend, saveDataToUrl()

  type Request = {
    start: number
    promise: Promise<any>
  }
  // let requests: Request[] = []
  const requests = new Map<number, Request>()
  let report = []

  const go = async () => {
    const urls = urlsText.split('\n').map((url) => {
      const u = new URL(url, baseUrl)
      return u.href
    })

    requests.clear()
    report = []
    let id = 0
    while (id < Number(requestsToSend) || Number(requestsToSend) === 0) {
      const url = urls[Math.floor(Math.random() * urls.length)]
      console.log(url)

      await roomInQueue()

      const promise = sendRequest(id, url)

      id += 1
    }

    await Promise.all(requests)
    console.log('done')
  }

  const roomInQueue = async () => {
    while (requests.size >= simultaneousRequests) {
      await new Promise((resolve) => setTimeout(resolve, 1))
    }
  }

  const sendRequest = async (id: number, url: string) => {
    const options: RequestInit = {
      mode: 'no-cors',
    }
    const start = Date.now()
    const promise = fetch(url, options)
      .then((response) => requestDone(id, response, null))
      .catch((error) => requestDone(id, null, error))

    requests.set(id, {
      start,
      promise,
    })
  }

  const requestDone = async (id: number, res: Response | null, err: null) => {
    const end = Date.now()
    const request = requests.get(id)
    requests.delete(id)

    console.log('requestDone', id)
    if (res && request) {
      await res.text()
      const duration = end - request.start
      console.log(duration)
    }
  }

  const clear = () => {
    urlsText = ''
    baseUrl = ''
  }
</script>

<main>
  <h1>Speed test</h1>

  <div class="test-input">
    <p>Base URL</p>
    <input bind:value={baseUrl} />
    <p>Urls</p>
    <textarea bind:value={urlsText}></textarea>
    <p>Requests to send (0 for continious)</p>
    <input bind:value={requestsToSend} />
    <p>Simultaneous requests: {simultaneousRequests}</p>
    <p>Stats</p>
    <p>Log</p>

    <p>
      <button type="button" on:click={go}>Go</button>
      <button type="button" on:click={clear}>Clear</button>
    </p>
  </div>
</main>

<style>
  .test-input {
    display: flex;
    flex-direction: column;
    align-items: left;
    justify-content: left;
    margin-bottom: 2rem;

    & p {
      text-align: left;
    }

    & textarea {
      height: 10rem;
    }
  }
</style>
