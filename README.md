# Next Level Vue

**Axios Interceptors** é usado para interceptar chamadas de API, onde iremos usar para pegar o status da chamada e atualizar a progress bar

Para executarmos isto podemos fazer de três maneiras diferentes.

## Axios Interceptors

```js
// EventService.js
import axios from "axios";
import NProgress from "nprogress";

const apiClient = axios.create({
  baseURL: `http://localhost:3000`,
  withCredentials: false, // This is the default
  headers: {
    Accept: "application/json",
    "Content-Type": "application/json"
  }
});

apiClient.interceptors.request.use(config => {
  NProgress.start();
  return config;
});
apiClient.interceptors.response.use(response => {
  NProgress.done();
  return response;
});

export default {
  getEvents(perPage, page) {
    return apiClient.get("/events?_limit=" + perPage + "&_page=" + page);
  },
  getEvent(id) {
    return apiClient.get("/events/" + id);
  },
  postEvent(event) {
    return apiClient.post("/events", event);
  }
};
```

Existem 2 ressalvas com esta solução:

- Não é otimizado para multiplas chamadas de API ao mesmo tempo, podendo ser resolvido através de um modulo Vuex para loading (loading.js)
- Template é carregado antes da chamada do callback before API ser chamado

OBS: **Interceptors são interessantes de conhecer pois podem adicionar tokens de autorização, formatar ou filtrar dados recebidos antes de chegar na aplicação, pegar respostas 401 not authorized responses**
