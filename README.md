> 在开始学习本教程之前，我们假定你已经能够熟练使用Generator、redux和react。

redux-saga 是一个在 React/Redux 应用中，可以优雅地处理 side effect(副作用：异步等)的库，你可以在一个地方创建sagas来专门处理side effect。
## 一个简单的例子
在store中引入中间件`redux-saga`
```javascript
//store
import { createStore, applyMiddleware } from 'redux';
import createSagaMiddleware, { runSaga } from 'redux-saga';
import reducer from './reducers';
import saga from './sagas';

const sagaMiddleware = createSagaMiddleware();
const store = createStore(reducer, applyMiddleware(sagaMiddleware));
sagaMiddleware.run(saga);

//sagas.js
function* helloSagas(){
  console.log('hello saga');
}

export default helloSagas
```
运行程序，就可以在控制台看到`hello saga`。现在让我们的saga开始捕获`action`，下面是一个简单的`Counter`组件,对于这种同步的action，我们无需借助saga，但是当我们点击按钮时，saga仍然能够监听到action的执行。运行下面的程序，当点击按钮时，在控制台会打印相应的action，但action并不是由saga来调用的。
```js
// component
import React, { Component } from 'react'
import {connect} from 'react-redux' 
import * as actions from '@/actions'
import { getSagaCounter } from '@/reducers'

class Counter extends Component {
  render() {
    const { increment, decrement } = this.props
    return (
      <div>
        <button onClick={increment}>increment</button>
        <button onClick={decrement}>decrement</button>
        <br />
        { this.props.counter }
      </div>
    )
  }
}

export default connect(
  (state) => ({counter: getSagaCounter(state)}),
  actions
)(Counter)

//reducer
import { combineReducers } from 'redux';

const sagaCounter = (state = 0, action) => {
  switch(action.type){
  case 'INCREMENT':
    return state + 1
  case 'DECREMENT':
    return state - 1
  default:
    return state
  }
}

export default combineReducers({
  sagaCounter
})

export const getSagaCounter = state => state.sagaCounter

//action
export const increment = () => ({
  type: 'INCREMENT'
})

export const decrement = () => ({
  type: 'DECREMENT'
})

//saga
import {takeEvery} from 'redux-saga'
function* listenAction(action){
  console.log(action);
}

function* incrementSagas(){
  //在每次 dispatch `INCREMENT` action 時，执行listenAction。
  yield takeEvery('INCREMENT', listenAction)
  //在每次 dispatch `DECREMENT` action 時，执行listenAction。
  yield takeEvery('DECREMENT', listenAction)
}

export default incrementSagas;
```
现在我们对做一点小改动，定义一个新的`action`:`MY_INCREMENT`，这个`action`的作用就是执行原来的`INCREMENT`，我们新添加一个按钮，点击时`dispatch`这个`action`

```js
const { increment, decrement, myIncrement } = this.props
return (
  ...
  <button onClick={myIncrement}>myIncrement</button>
  ...
)
//action
export const myIncrement = () => ({
  type: 'MY_INCREMENT'
})
//sagas
import {takeEvery, put} from 'redux-saga/effects'
function* increment(action){
  yield put({type:'INCREMENT'})//dispatch
}

function* incrementSagas(){
  yield takeEvery('MY_INCREMENT', increment)
}

export default incrementSagas;
```
当saga拿到`MY_INCREMENT`时，执行`increment`函数，而`increment`函数就是`dispatch INCREMENT`，从而实现的counter的增加。此时我们就发现，如果要MY_INCREMENT延迟执行的话，只需要在`increment`函数中yeild一个延迟函数,当一个Promise在saga中yield时，saga将暂停执行，知道这个Promise被resolve，然后saga继续执行。
```js
const delay = ms => new Promise(resolve => setTimeout(resolve, ms))

function* increment(action){
  yield delay(1000) //延迟1s
  yield api.fetchSomeData() // 等待数据返回
  yield put({type:'INCREMENT'}) //上面的都完成后，dispatch `INCREMENT` action
}
```
现在回过头，看看我们最开始的helloSagas，如果此时要在加上这个saga该如何实现呢，在`redux-saga/effects`里面有一个all方法，只需要将多个saga放入all里面就可以了，因此很容易实现saga文件的拆分
```js
export default function* rootSaga() {
  yield all([
    helloSaga(),
    incrementSagas()
  ])
}

```
到这里，我们大致已经了解了saga的基本原理，后面的教程将通过一个todolist详细介绍具体的细节。
上面的源码[点这里](https://github.com/rainke/saga-tutorial)

## 准备工作
* 一个数据库：
![](database.jpeg)

* 安装：react react-dom react-redux redux redux-saga react-router-dom react-router axios

* node[服务器](https://github.com/rainke/saga-tutorial/tree/master/app)

最开始的代码就是这个样子:
```js
//App.js
import React from 'react'
import {BrowserRouter} from 'react-router-dom'
import TodoList from './components/TodoList'
const App = () => (<div>
  <BrowserRouter>
    <div>
      <h1>Hello Saga</h1>
      <div>
        <TodoList />
      </div>
    </div>
  </BrowserRouter>
</div>)

export default App

//TodoList.js
import React from 'react'
import AddTodo from './AddTodo'
import VisibleTodoList from './VisibleTodoList'
import Footer from './Footer'


const TodoList = () => (
  <div>
    <AddTodo />
    <VisibleTodoList />
    <Footer />
  </div>
)

export default TodoList

//AddTodo.js
import React from 'react'

const AddTodo = () => {
  let input
  return (
    <div>
      <input ref={node => {
          input = node
        }} />
        <button>add todo</button>
    </div>
  )
}

export default AddTodo

//VisibleTodoList.js
import React from 'react'
const VisibleTodoList = () => (
  <div>
    <p>
      <span>todo</span>
      <a href="#">×</a>
    </p>
  </div>
)

export default VisibleTodoList

//Footer
import React from 'react';

const Footer = () => (
  <p>
    <a href="#">all</a>
    {' '}
    <a href="#">active</a>
    {' '}
    <a href="#">completed</a>
  </p>
)

export default Footer
```
现在，让我们来做第一件事，请求todos列表。需要想服务端发起请求，这里需要三个actions，首先是发起请求，然后是请求成功和请求失败。
```JS
export const fetchTodo = (filter) => ({
  type:'FETCH_TODO',
  filter
})
export const fetchTodoSuccess= (todos) => ({
  type: 'FETCH_TODO_SUCCESS',
  todos
})

export const fetchTodoError= () => ({
  type: 'FETCH_TODO_ERROR'
})
```
我们的列表只会在`FETCH_TODO_SUCCESS`的时候才会改变，因此相应的reducers就会想这样，我们每一个reducer都要暴露一些方法用于获取store中的数据，在component中使用。
```js
const todos = (state = [], action) => {
  switch(action.type){
  case 'FETCH_TODO_SUCCESS':
    return action.todos
  default:
    return state
  }
}
export const getTodos = (state) => state.todos
```
现在我们重构VisibleTodoList，使其能够重store中获取数据,并将数据显示在列表中。
```js
import React, { Component } from 'react'
import { connect } from 'react-redux'
import { fetchTodo } from '@/actions'
import {getTodos} from '@/reducers'
class VisibleTodoList extends Component {
  componentDidMount(){
    this.props.fetchTodo();
  }
  render() {
    const { todos } = this.props
    return  (
      <div>
        {todos.map(todo => (
          <p key={todo.id}>
            <span>{todo.text}</span>
            <a href="#">×</a>
          </p>
        ))}
      </div>
    )
  }
}

export default connect(
  (state) => ({
    todos: getTodos(state)
  }),
  { fetchTodo }
)(VisibleTodoList)
```

通过connect向组件的props传入`todos`值和`fetchTodo`方法，`fetchTodo`方法在组件装载完毕后调用，而此时，我们的saga监听到这个action，并对其做相应的处理
```js
import {takeEvery, put} from 'redux-saga/effects'
import * as api from '@/api'

function* fetchTodo(action){
  const {todos, error} = yield api.getTodos()
  if(todos){
    yield put({type:'FETCH_TODO_SUCCESS', todos})
  } else {
    yield put({type: 'FETCH_TODO_ERROR'})
  }
}

function* fetchTodoSaga(){
  yield takeEvery('FETCH_TODO', fetchTodo)
}

//api
export const getTodos = (params) => 
  axios.get('/api/list', {params})
    .then(result => ({todos: result.data}))
    .catch(err => ({err}))
```
当saga监听到`FETCH_TODO`触发，就执行`fetchTodos`函数,函数中向服务器发起请求，如果成功得到数据后，dispatch `FETCH_TODO_SUCCESS`，否则执行dispatch `FETCH_TODO_ERROR`

此时我们能够从浏览中看到我们从数据库中取得的数据，现在，我们要对`VisibleTodoList`组件做优化，使其与`Footer`组件配合，能够根据footer的状态显示相应的数据，这里通过引入react-router来实现，这里直接给代码: [源码](https://github.com/rainke/saga-tutorial/tree/3073b7d67ca0d5c28f02989335f49b6d9c83c957)
```js
import {takeEvery, put} from 'redux-saga/effects'
import * as api from '@/api'

function* fetchTodo(action){
  const {todos, error} = yield api.getTodos({filter:action.filter})
  if(todos){
    yield put({type:'FETCH_TODO_SUCCESS', todos})
  } else {
    yield put({type: 'FETCH_TODO_ERROR'})
  }
}

function* fetchTodoSaga(){
  yield takeEvery('FETCH_TODO', fetchTodo)
}

export default fetchTodoSaga
```
现在我们仔细揣测下这个saga，每当收到`FETCH_TODO`就执行请求，返回后就分发`FETCH_TODO_SUCCESS`，然而这个请求不可控，当我们快速切换filter时，连续发起多个请求，此时页面显示的结果是最后执行`FETCH_TODO_SUCCESS`的结果，但这可能跟我们预想的不一样。正确的做法是，每当发起一个`FETCH_TODO`就取消上一次，让结果永远是最新的`FETCH_TODO`的结果。因此我们不能采用`takeEvery`，需要替换成`takeLatest`，其作用就是执行最新的`FETCH_TODO`时会取消上一次的`FETCH_TODO`，即上一次请求返回后不会再dispatch其他action。
