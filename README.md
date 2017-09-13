# redux

## action

actions目录下存放的是Action Creator

### 普通Action

即普通Redux中的Action对象，type为enum常量
``````javascript
{
    type:'agent/init-current-agent',
    payload,
    next,
  }
``````

### WebRequest Action

以type为`webRequest/`开头的Action
``````javascript
{
    type: `${webRequest}/${type}`,
    payload,
    next,
  }
``````
这类Action在经过MiddleWare `store/web/index.js`中时会进行拦截，然后做如下操作：
* 通过`adapter`方法转换成commands命令
```javascript
const commands = adapter(store.getState(), action);
```
* 调用`queue`或`longTimeQueue`对commands进行dispatch

之后请求进入队列等待发送

### WebResponse Action

请求在发送之后返回结果，通过queue里subscribe的output（store/web/output.js）方法将response重新组成action（resonpse中带有请求中的type）
并使用webRequest刚刚拦截时的next对象dispatch出来以继续传递下去
所以在reducers中具有一些如下类型的处理器
```javascript
[`${webResponse}/${chatAction.agentSendEmail}`](state) {
    return updateIn(state, 'mychats', 'sendEmailSuccessMessage', 'Email sent successfully.');
  }
```

### WebError Action

WebError 这类Action主要是用于处理一些请求时造成的异常信息
```javascript
[`${webError}/${chatAction.agentExtendChargeDate}`](state, action) {
    return Object.assign({}, state, {
      login: Object.assign({}, state.login, {
        ifShowErrorMessage: true,
        ifExtendLoading: false,
        errorMessage: action.payload.errorString || 'error',
      }),
    });
  },
```

## reducer

### selectors （get）

selectors主要是定义一些选择器api，便于从state中获取想要的数据
```javascript
export const getCurrentAgent = state => state.agent;
export const getCurrentAgentId = state => state.agent.id;
export const getCurrentAgentName = state => state.agent.name;
export const getCurrentAgentEmail = state => state.agent.email;
export const getCurrentAgentStatus = state => state.agent.status;
export const getCurrentAgentSession = state => state.agent.session;
export const getCurrentAgentPassword = state => state.agent.password;
```

### reducer（set）

reducer作用是根据旧的state将改变并入其中，然后返回一个新的state对象
`reducers/selectors/reduceReducers.js`中的`reduceReducers`方法用于将reducers进行combine，之所以不用redux原生的`combineReducers`是因为特殊需求导致的，除了`partState`与`action`之外，还需要提供一个`totalState`参数。
最后`reduceReducers`会返回一个整体的`reducer(state,action)`

## containers

containers内部主要是react-redux的container组件，用于对react组件进行业务逻辑包装，而react则保持纯组件性质，不参与业务逻辑处理
connect方法的三个参数为：
`mapStateToProps`
`mapDispatchToProps`
`mergeProps`
mapStateToProps作用是通过state对象映射成Props属性传入React组件
mapDispatchToProps作用是定义Props中的行为回调dispatch(action)，通过返回具备回调函数的对象，传入React组件

## store

store通过redux的`createStore`方法进行生成
```javascript
const store = createStore(
    reducer,
    initialState,
    enhancer
  );
```
* reducer通过`reducers/index.js`生成
* initialState为空，可以同步服务器状态
* enhancer则为中间件，通过redux的`applyMiddleware`生成
```javascript
middleware = applyMiddleware(
      promise,
      web,
      chatFilter,
      notification,
      globalError,
      windowsManager,
    );
```

### middleware

中间件写法如下：
```javascript
//代码段1
export default store => next => (action) => {
	return next(action);
}
```
首先，`applyMiddleware`调用时会传入变长参数（middlewares)，返回一个方法，
```javascript
//代码段2
return (createStore) => (reducer, preloadedState, enhancer) => {
	......
	return {
    	...store,
        dispatch
    }
}
```
用以传入`createStore`，然后在`createStore`方法中，判断如果存在`enhancer`则将`自身`传入以上方法，得到以上方法的里面一层
```javascript
//代码段3
(reducer, preloadedState, enhancer) => {
	......
	return {
    	...store,
        dispatch
    }
}
```
然后将reducer,preloadedState（也就是initialState）传入，进入......（截图中有这部分逻辑）进行处理：
* 将入参传入调用createStore，得到一个store对象，此处enhancer对象应该为空了；
* 暂存store的dispatch方法，因为之后要对其进行重写；
* 准备一个middlewareAPI对象，具备getState与dispatch方法（**所以中间件中的store并不是一个完整的store对象，而是模拟的**）；
* 之后一步是重点，对middlewares进行遍历，传入模拟store进行调用，返回middleware的第二层函数
```javascript 
//代码段4
next => (action) => {
	return next(action);
}
```
得到一个chain列表（等于剥掉了中间件写法的第一层）；
* 对chain列表进行`compose`：compse方法的作用是将这些类似过滤器的中间件串起来，比如compose(a,b,c,d,e)得到的调用结果是：`(...args)=>a(b(c(d(e(...args))))`;
```javascript 
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
```
* 最后将store.dispatch传入上面得到的函数，最先调用的是e(store.dispatch)，通过`代码段4`得到一个结果
`(action)=>{ return next(action);}`，其中这个`next`就是传入的`store.dispatch`，再将这个结果传入d(...args)，再将结果传入c,b,a...最后从a函数中返回一个`(action)=>{ return next(action);}`，这里面的`next`则是b中返回的。于是最后的这个dispatch方法行成了一个调用顺序，从a->b->c->d->e，最后才调用store原本的dispatch方法。**此方法的作用在于对原本的dispatch方法做一个代理，在真正执行dispatch之前，经过一系列的中间件的处理**。




## components

React纯组件，如果能用纯组件写法就用纯组件写法，尽量以性能最优的形式来写，因为业务逻辑全放在Container中去了。
