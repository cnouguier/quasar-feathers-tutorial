# Combining the power of [Quasar](http://quasar-framework.org/) and [Feathers](http://feathersjs.com/) to build real-time web apps

A couple of days ago I 've started looking at what could be the foundation to build good-looking real-time web apps for new projects. Starting from a [Angular 1.x](https://angularjs.org/) background I was looking for something based on new Javascript standards (TypeScript, ES2015, ES2016, etc.), lightweight and easy to learn, as well as having less to do with the big players. I found [Vue.js](https://vuejs.org/) that mostly satisfy all these criteria. However, it missed a built-in component library, which brings me naturally to find [Quasar](http://quasar-framework.org/). 

Then I looked for something similar to handle the most basic tasks of creating real-time web apps on the server-side, I dreamed of a framework handling indifferently REST/socket API calls, with built-in support for most authentication schemes, being database/transport agnostic so that I could develop microservices powering different technologies. I naturally found [Feathers](https://blog.feathersjs.com/introducing-feathers-2-0-aae8ae8e7920), which additionnaly provides all of this with a plugin based architecture around a minimalist core.

I decided to start building a basic real-time chat app inspired from https://github.com/feathersjs/feathers-chat :
[![Authentication video](https://img.youtube.com/vi/_iqnjpQ9gRo/0.jpg)](https://www.youtube.com/watch?v=_iqnjpQ9gRo)
[![Chat video](https://img.youtube.com/vi/te1w33vaDXI/0.jpg)](https://www.youtube.com/watch?v=te1w33vaDXI)

## Disclaimer

Although this tutorial details the path to create an application skeleton featuring Quasar and Feathers from scratch, as well as code details, most of this work is currently under integration in the Quasar ecosystem. Indeed, Quasar provides the **wrapper** concept which allows to plug the frontend app into a larger piece of work such as Electron or Express powered backend. The simplest way to retrieve and start with this application skeleton is to use the Quasar Feathers wrapper guide https://github.com/quasarframework/quasar-wrapper-feathersjs-api.

Last but not least, I assume your are familiar with the [Vue.js](https://vuejs.org/) and [Node.js](https://nodejs.org) ecosystem.

## Installation and configuration

Each framework provides its own CLI so that starting a project is easy, with a couple of instructions you have everything ready to start coding your app.

Quasar for the frontend:
```bash
$ npm install -g quasar-cli
$ quasar init quasar-chat
$ cd quasar-chat
$ npm install
// Will launch the frontend server in dev mode on 8080
$ quasar dev
```
Feathers for the backend:
```bash
$ npm install -g feathers-cli
$ mkdir feathers-chat
$ cd feathers-chat
// Use defaults
$ feathers generate
// Will launch the backend server in dev mode on 3030
$ npm start
```

Because we generated the Feathers boilerplate with authentication we already have a **user** service providing [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations as well. But as we want to develop a chat application we miss a **message** service so we generate it in the backend folder:
```bash
feathers generate service
```

To make the Quasar app correctly contacting the backend you have to configure an API proxy in your frontend **config/index.js**:
```javascript
...
  dev: {
    proxyTable: {
      '/api': {
        target: 'http://localhost:3030',
        changeOrigin: true
      }
    }
    ...
```

## API glue

Feathers provides you with a thin layer on the client-side to make API authentication and calls so simple. We create a new **api.js** file in the frontend to handle the glue with the API:
```javascript
import feathers from 'feathers'
import hooks from 'feathers-hooks'
import socketio from 'feathers-socketio'
import auth from 'feathers-authentication-client'
import io from 'socket.io-client'

const socket = io('http://localhost:3030', {transports: ['websocket']})

const api = feathers()
  .configure(hooks())
  .configure(socketio(socket))
  .configure(auth({ storage: window.localStorage }))

api.service('/users')
api.service('/messages')

export default api
```

Now the API is easy to integrate in any component to perform the various tasks we need, e.g.:
```javascript
import api from 'src/api'
const users = api.service('users')
// Authenticate
api.authenticate({
  strategy: 'local',
  email: email,
  password: password
}).then(user => {
  Toast.create.positive('Authenticated')
})
// Get all users
users.find().then((response) => {
  this.$data.users = response.data
})
// Liste to user events
users.on('created', user => {
  this.$data.users = this.$data.users.concat(user)
})
```

## Main layout

From a end-user perspective the application will be simple:
 - a menu toolbar including (**Index.vue** component)
   - a sign in/register entry when not connected
   - home/chat entries and a profile menu to logout when connected
 - a sidebar menu recalling the home/chat entries and a about section
 - a landing home page displaying different text depending on the connection state (**Home.vue** component)
 - a signin/register form with email/password (**SignIn.vue** component)
 - a chat view listing available users and providing real-time messages read/write (**Chat.vue** component)
 
 The main app layout is already part of the Quasar default template so we will directly modify it but additional components can be generated using the CLI:
 ```bash
 $ quasar new Home
 $ quasar new SingIn
 $ quasar new Chat
 ```
 
 We update the layout of the **Index.vue** template to include a [Toolbar with some entries](http://quasar-framework.org/components/toolbar.html), a profile menu with a logout entry using a [Floating Action Button](http://quasar-framework.org/components/floating-action-buttons.html), a [Sidebar menu](http://quasar-framework.org/components/drawer.html) and an [entry point for other components](https://router.vuejs.org/en/api/router-view.html):
 ```html
 <q-layout>
    <div slot="header" class="toolbar">
      <button @click="$refs.menu.open()" v-show="authenticated">
        <i>menu</i>
        <q-tooltip anchor="bottom middle" self="top middle" :offset="[0, 20]">Menu</q-tooltip>
      </button>
      <q-toolbar-title :padding="0">
        Quasar + Feathers boilerplate
      </q-toolbar-title>
      <button class="primary" @click="goTo('signin')" v-show="!authenticated">
        Sign In
      </button>
      ...
      <q-fab icon="perm_identity" direction="left" v-show="authenticated">
        <q-small-fab class="primary" @click.native="signout" icon="exit_to_app">
          <q-tooltip anchor="bottom middle" self="top middle" :offset="[0, 20]">Sign Out</q-tooltip>
        </q-small-fab>
      </q-fab>
    </div>

    <q-drawer swipe-only left-side ref="menu" v-show="authenticated">
      <div class="toolbar light">
        <i>menu</i>
        <q-toolbar-title :padding="1">
            Menu
        </q-toolbar-title>
      </div>
      <q-drawer-link icon="home" to="/chat">Home</q-drawer-link>
      <q-drawer-link icon="chat" to="/chat">Chat</q-drawer-link>
      <q-collapsible icon="info" label="About">
        <p style="padding: 25px;" class="text-grey-7">
          This is a template project combining the power of Quasar and Feathers to create real-time web apps.
        </p>
      </q-collapsible>
    </q-drawer>

    <!-- sub-routes -->
    <router-view class="layout-view" :user="user"></router-view>
    
  </q-layout>
 ```
 
 We update the router configuration in **router.js** to reflect this as well:
 ```javascript
 routes: [
    {
      path: '/',
      component: load('Index'),
      children: [
        {
          path: '/home',
          name: 'home',
          component: load('Home')
        },
        {
          path: '/signin',
          name: 'signin',
          component: load('SignIn')
        },
        {
          path: '/register',
          name: 'register',
          component: load('SignIn')
        },
        {
          path: '/chat',
          name: 'chat',
          component: load('Chat')
        }
      ]
    },
    {
      path: '*',
      component: load('Error404')
    } // Not found
  ]
 ```
 
## Authentication

### Backend

### Frontend

