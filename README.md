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

## Global and Per-Route Guards

Ultima solução a ser vista (e escolhida).

Existem mais 2 **Global Routing Guards** que devemos conhecer, mas para utilizarmos eles, devemos adicionar algumas configurações

```js
Vue.use(Router)
// Guards são chamados no objeto router
// pois são chamados usando router.nomeDoGuard()
const router = new Router({ ... })
```

Os novos life cycles são

```js
// Roda antes de navegar a um componente
// Precisar chamar next()
router.beforeEach(routeTo, routeFrom, next);
// Roda já depois do componente ser criado
// Não precisar chamar next()
router.afterEach(routeTo, routeFrom, next);
```

A order deles serem chamados é a seguinte:

```js
beforeEach();
beforeEnter();
beforeRouteEnter();
afterEach();
beforeCreate();
created();
// Outros components life cycles
```

### Implementação

```js
// router.js
import Vue from "vue";
import Router from "vue-router";
import EventCreate from "./views/EventCreate.vue";
import EventList from "./views/EventList.vue";
import EventShow from "./views/EventShow.vue";
import NProgress from "nprogress";
import store from "@/store/store";

Vue.use(Router);

const router = new Router({
  mode: "history",
  routes: [
    {
      path: "/",
      name: "event-list",
      component: EventList
    },
    {
      path: "/event/create",
      name: "event-create",
      component: EventCreate
    },
    {
      path: "/event/:id",
      name: "event-show",
      component: EventShow,
      props: true,
      beforeEnter(routeTo, routeFrom, next) {
        store.dispatch("event/fetchEvent", routeTo.params.id).then(() => {
          next();
        });
      }
    }
  ]
});

router.beforeEach((routeTo, routeFrom, next) => {
  NProgress.start();
  next();
});
router.afterEach(() => {
  NProgress.done();
});

export default router;
```

```js
// event.js
fetchEvent({ commit, getters, dispatch }, id) {
  var event = getters.getEventById(id)

  if (event) {
    commit('SET_EVENT', event)
  } else {
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
//
// Chamadas estão sendo realizadas no router
// então o componente nao precisa fazer mais
// import { mapState, mapActions } from "vuex";
import { mapState } from "vuex";

export default {
  props: ["id"],
  // created() {
  //   this.fetchEvent(this.id)
  // },
  computed: mapState({
    event: state => state.event.event
  })
  // methods: mapActions('event', ['fetchEvent'])
};
```

Ao utilizar este modo, todo código relacionado ao Vuex agora está fora do componente, sendo mantido agora no arquivo de rotas. Com isto, podemos remover o Vuex do componente passando o valor como uma prop ao componente.

```html
<!-- EventShow.vue -->
<template>
  <div>
    <div class="event-header">
      <span class="eyebrow">@{{ event.time }} on {{ event.date }}</span>
      <h1 class="title">{{ event.title }}</h1>
      <h5>Organized by {{ event.organizer ? event.organizer.name : '' }}</h5>
      <h5>Category: {{ event.category }}</h5>
    </div>

    <BaseIcon name="map">
      <h2>Location</h2>
    </BaseIcon>

    <address>{{ event.location }}</address>

    <h2>Event details</h2>
    <p>{{ event.description }}</p>

    <h2>
      Attendees
      <span class="badge -fill-gradient"
        >{{ event.attendees ? event.attendees.length : 0 }}</span
      >
    </h2>
    <ul class="list-group">
      <li
        v-for="(attendee, index) in event.attendees"
        :key="index"
        class="list-item"
      >
        <b>{{ attendee.name }}</b>
      </li>
    </ul>
  </div>
</template>

<script>
  // import { mapState } from 'vuex'

  export default {
    props: {
      event: {
        type: Object,
        required: true
      }
    }
    // computed: mapState({
    //   event: state => state.event.event
    // })
  };
</script>

<style scoped>
  .location {
    margin-bottom: 0;
  }
  .location > .icon {
    margin-left: 10px;
  }
  .event-header > .title {
    margin: 0;
  }
  .list-group {
    margin: 0;
    padding: 0;
    list-style: none;
  }
  .list-group > .list-item {
    padding: 1em 0;
    border-bottom: solid 1px #e5e5e5;
  }
</style>
```

```js
// router.js
import Vue from "vue";
import Router from "vue-router";
import EventCreate from "./views/EventCreate.vue";
import EventList from "./views/EventList.vue";
import EventShow from "./views/EventShow.vue";
import NProgress from "nprogress";
import store from "@/store/store";

Vue.use(Router);

const router = new Router({
  mode: "history",
  routes: [
    {
      path: "/",
      name: "event-list",
      component: EventList
    },
    {
      path: "/event/create",
      name: "event-create",
      component: EventCreate
    },
    {
      path: "/event/:id",
      name: "event-show",
      component: EventShow,
      props: true,
      // Por ter props true podemos enviar
      // o resultado da action como props
      beforeEnter(routeTo, routeFrom, next) {
        store.dispatch("event/fetchEvent", routeTo.params.id).then(event => {
          routeTo.params.event = event;
          next();
        });
      }
    }
  ]
});

router.beforeEach((routeTo, routeFrom, next) => {
  NProgress.start();
  next();
});
router.afterEach(() => {
  NProgress.done();
});

export default router;
```

```js
// event.js
fetchEvent({ commit, getters, dispatch }, id) {
  var event = getters.getEventById(id)

  if (event) {
    commit('SET_EVENT', event)
    // Retornar o evento da action
    return event
  } else {
    return EventService.getEvent(id)
      .then(response => {
        commit('SET_EVENT', response.data)
        // Retornar o evento da action
        return response.data
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

Com isto concluimos as alterações para tirar a dependencia Vuex do componente EventShow

## Base components

Como funciona a diretiva v-model

```js
// v-model="event.title"
:value="event.title"
@input="(value) => { event.title = value }"
```

Quando passamos parâmetros html para um componente, ele será adicinado automaticamente na tag root deste componente.
Os atributos que podem acontecer isto por exemplo são type e placeholder.

Para receber os atributos em outro lugar, usamos o seguinte comando no componente:

```js
export default {
  inheritAttrs: false
};
```

E adicionamos o seguinte atributo na tag para adicionar no local onde queremos:

```html
<div v-bind="$attrs"></div>
```
