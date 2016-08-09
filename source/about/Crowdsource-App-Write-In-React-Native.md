---
title: Crowdsource App Write In React-Native
date: 2016-08-02 15:02:53
---

> This article discusses about how we compose React-Native-based components with Redux in Hummingbird Crowdsource App.

## Introduction

It is easy to write a simple app with `React Native`, because at the beginning you do not care about the deeper problems such as the data flow and possible extension in the future. However, as the amount of components written by React Native increases, it is of vital importance to consider these problems because the system becomes larger and its coupling becomes higher.

![](http://7o4zmy.com1.z0.glb.clouddn.com/react.png)

Among all the techniques to decrease the coupling of a system, `Redux` is, in our opinion, a better choice. As is known, `React Native` renders the UI according to the **“state”** and the **“props”**. But it is harder to maintain the consistency between the UI and the  **“state”** or the **“props”** as the amount of components increases. To solve this problem, the `Flux` architecture provides an elegant pattern for controlling the data flow and rendering the UI, which can be introduced to perform the navigation in `React Native`. And `Redux` is a predictable state container for React Native, which evolves the ideas of Flux. Therefore, in the React Native development of our app, we use Redux to control the data flow, to decrease the coupling between each component and to make it easier to add more components into our project.

Now let’s discuss the details.

## React Native with Redux

Now I give you an example to illustrate how we control the data flow in our crowdsource project using the `Redux` architecture. In the app of our project, a user may sometimes inquire his member level. There is, of course, a corresponding component in our app which provides him with such information. The UI of the component shows the user’s member level. When the user enters the component, a request is sent to fetch the member level from the server and after getting the result of the request, the UI is refreshed to show the member level.

First of all, we define the `ActionType` for fetching the member level from the server:

```
export const FETCH_RANK_LIST = 'FETCH_RANK_LIST';

```
Then we define the corresponding Action:
```javascript
'use strict';

import * as types from '../constants/ActionTypes';
import {LEVEL_PRIVILEGES} from '../constants/Urls';
import {request} from '../utils/RequestUtils';
import {ToastShort} from '../utils/ToastUtils';

export function fetchLevelPrivileges() {
	return dispatch => {
		dispatch(fetchRankList());
		request(LEVEL_PRIVILEGES, 'get')
			.then((rankList) => {
				dispatch(receiveRankList(rankList));
			})
			.catch((error) => {
				dispatch(receiveRankList([]));
				if (error != null) {
					ToastShort(error.message)
				}
			})
	}
}

function fetchRankList() {
	return {
		type: types.FETCH_RANK_LIST,
	}
}

function receiveRankList(rankList) {
	return {
		type: types.RECEIVE_RANK_LIST,
		rankList: rankList
	}
}
```
The Action here is asynchronous since the request is asynchronous. As is shown above, we use the function fetchLevelPrivileges to fetch the member level from the server and after receiving the data, we trigger the Action **(RECEIVE_RANK_LIST)** for receiving the data.
After the definition of the Action, we define the `Reducer`.

```javascript
'use strict';

import * as types from '../constants/ActionTypes';

const initialState = {
	loading: false,
	rankList: []
}

export default function rank(state = initialState, action) {
	switch (action.type) {
		case types.FETCH_RANK_LIST:
			return Object.assign({}, state, {
				loading: true
			});
		case types.RECEIVE_RANK_LIST:
			return Object.assign({}, state, {
				loading: false,
				rankList: action.rankList
			})
		default:
			return state;
	}
}
```
The state is initialState at the beginning and is then updated according to different types. After each update operation, we get a new object of the `state` rather than the original object with some fields updated.

Now we define the Store.
```javascript
'use strict';

import {createStore, applyMiddleware} from 'redux';
import thunkMiddleware from 'redux-thunk';
import rootReducer from '../reducers/index';

const createStoreWithMiddleware = applyMiddleware(thunkMiddleware)(createStore);

export default function configureStore(initialState) {
	const store = createStoreWithMiddleware(rootReducer, initialState);

	return store;
}
```

We use `redux-thunk` to support asynchronous Actions.
rootReducer is the combined `reducer` in the end.
```javascript
'use strict';

import {combineReducers} from 'redux';
import notice from './notice';
import rank from './rank';

const rootReducer = combineReducers({
	notice,
	rank
})

export default rootReducer;
```
We use the function combineReducers to combine multiple reducers into one.

Finally we do the following to make the data flow work.
```javascript
import React from 'react-native'
import {Provider} from 'react-redux/native'
import configureStore from './store/configure-store'

import App from './containers/app'

const store = configureStore();

class Root extends React.Component {
	render() {
		return (
			<Provider store={store}>
        {() => <App />}
      </Provider>
		)
	}
}

export default Root;
```
This is the only registered entrance in `index.android.js`. It uses the Provider to inject the store into the app.

## summary

Now a summary of the data flow is given below:
When the user goes into the component of the member level, the component uses the function fetchLevelPrivileges to fetch the member level from the server. Meanwhile the **FETCH_RANK_LIST** action is dispatched, the reducer updates the state and the UI is then refreshed to show a loading dialog.
After getting the result from the server, the **RECEIVE_RANK_LIST** action is dispatched, the reducer updates the state and the UI is then refreshed to show the member level.
Therefore, the data flows according to such a pattern: `Action`=>`Dispatch` => `Store` => `View`. Whenever the user touches a button to perform an operation, an action is dispatched from the View and the data flows according to the pattern. 

The above is how we use the Redux architecture in our project. In this way, we make our project more flexible and easier to be extended.
