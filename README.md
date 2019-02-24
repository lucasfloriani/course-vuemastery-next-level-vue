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

## In-Component Route Guards

Outra maneira de realizar uma progress bar com base na resposta de chamada da API

Normalmente quando utilizamos componentes Vue, nos podemos utilizar os seguintes lifecycles:

```js
beforeCreate();
created();
beforeMount();
mounted();
beforeUpdate();
updated();
beforeDestroy();
destroyed();
```

Porem quando utilizamos o Vue Router, nós obtemos mais 3 hooks, os quais são chamados de **Route Navigation Guards**

```js
// Chamado antes do componente ser criado
// Sem acesso ao 'this' pq o componente não foi criado ainda
beforeRouteEnter(routeTo, routeFrom, next);
// Chamado quando a rota muda,porem ainda estamos usando o mesmo componente
// Tem acesso ao 'this'
beforeRouteUpdate(routeTo, routeFrom, next);
// Chamado quando o componente é navegado para longe
// Tem acesso ao 'this'
beforeRouteLeave(routeTo, routeFrom, next);
```

- **routeTo**: é a rota que esta sendo navegada
- **routeFrom**: É a rota que estamos saindo (atual)
- **next**: A função que precisar ser chamada para resolver o hook

OBS: Todos esses hooks precisam chamar o **next()**, onde quando chamado, confirma a navegação
OBS2: Passando o parâmetro false para o next, cancela a navegação
OBS3: Passando uma url ou nome da url, redireciona para o caminho passado

Exemplo:

```js
beforeRouteLeave(routeTo, routeFrom, next) {
  const answer = window.confirm(
    'Você realmente deseja sair? Existem pendências não salvas!'
  )
  if (answer) {
    next()
  } else {
    next(false)
  }
}
```

### Implementação

```js
// event.js
fetchEvent({ commit, getters, dispatch }, id) {
  var event = getters.getEventById(id)

  if (event) {
    commit('SET_EVENT', event)
  } else {
    // Agora precisa retornar uma promise
    return EventService.getEvent(id)
      .then(response => {
        commit('SET_EVENT', response.data)
      })
      .catch(error => {
        const notification = {
          type: 'error',
          message: 'There was a problem fetching event: ' + error.message
        }
        dispatch('notification/add', notification, { root: true })
      })
  }
}
```

```js
// EventShow.vue

// import { mapState, mapActions } from 'vuex'
// Removido pois usa o lifecycle beforeRouteEnter
import { mapState } from 'vuex'
import NProgress from 'nprogress'
import store from  '@/store/store

export default {
  props: ['id'],
  beforeRouteEnter(routeTo, routeFrom, next) {
    NProgress.start()
    store.dispatch('event/fetchEvent', routeTo.params.id)
      .then(() => {
        NProgress.done()
        next()
      })
  },
  // Removido pois usa o lifecycle beforeRouteEnter
  // created() {
  //   this.fetchEvent(this.id)
  // },
  computed: mapState({
    event: state => state.event.event
  }),
  // Removido pois usa o lifecycle beforeRouteEnter
  // methods: mapActions('event', ['fetchEvent'])
}
```
