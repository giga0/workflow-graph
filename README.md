# workflow-graph

### Project setup
```
yarn install
```

### Compiles and hot-reloads for development
```
yarn serve
```

### App uses JSON Server for fake API so it needs to be installed
```
yarn add json-server
```

### In the root folder of the app you will find db.json file, so to run server type:
```
json-server db.json
```

### Technologies used
- Webpack
- Vue
- Axios
- JSON Server

### Introduction
This app is used to draw the workflow graph fetched from API.
Currently app uses fake JSON Server for REST API for test purposes but in future will be connected with GraphQL API using Apollo.
