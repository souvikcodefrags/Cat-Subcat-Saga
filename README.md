# Cat-Subcat-Saga
Simple Category Subcategory with Redux Saga

### Folder Structure

* actions   
  * index.js  
  * types.js  
* api
  * index.js
* components
  * app.js
  * category-link.js
* reducers
  * category.js
  * index.js
* sagas
  * index.js
* utils
  * index.js
* index.html
* index.js
* package.json

***

### index.js
```sh
import React from "react";
import ReactDOM from "react-dom";
import { createStore, applyMiddleware } from "redux";
import { Provider } from "react-redux";
import createSagaMiddleware from "redux-saga";

import reducer from "./reducers";
import rootSaga from "./sagas";
import App from "./components/App";

const sagaMiddleware = createSagaMiddleware();
const store = createStore(reducer, applyMiddleware(sagaMiddleware));
sagaMiddleware.run(rootSaga);

ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById("root")
);
```

### /actions/index.js
```sh
export const Action = (type, data) => ({ type, data });
```

### /actions/types.js
```sh
export const FETCH_CATEGORIES = "FETCH_CATEGORIES";
export const FETCH_CATEGORIES_SUCCEEDED = "FETCH_CATEGORIES_SUCCEEDED";
export const FETCH_LINKS = "FETCH_LINKS";
export const FETCH_LINK_SUCCEEDED = "FETCH_LINK_SUCCEEDED";
export const RESET_LINKS = "RESET_LINKS";
export const SELECT_CATEGORY = "SELECT_CATEGORY";
```

### /api/index.js

```sh
import axios from "axios";

const Api = {
  fetchCategories: () => {
    const url = "https://api.publicapis.org/categories";
    return axios.get(url);
  },
  fetchLinks: category => {
    const url = `https://api.publicapis.org/entries?category=${encodeURIComponent(
      category
    )}&https=true`;
    return axios.get(url);
  }
};

export default Api;
```


### /components/app.js

```sh
import React from "react";
import PropTypes from "prop-types";
import { connect } from "react-redux";

import { emptyFunction } from "../utils";
import CategoryLink from "./CategoryLink";
import { Action } from "../Actions";
import {
  FETCH_CATEGORIES,
  FETCH_LINKS,
  RESET_LINKS,
  SELECT_CATEGORY
} from "../Actions/types";

export class App extends React.Component {
  componentDidMount() {
    const { fetchCategories } = this.props;
    fetchCategories();
  }

  onChangeCategory = e => {
    const { onChangeCategory } = this.props;
    const category = e.target.value;
    onChangeCategory(category);
  };

  render() {
    const { categories, categoryLinks, loadingLinks } = this.props;
    const hasCategoryLinks = !loadingLinks && categoryLinks.length > 0;
    return (
      <div>
        <h2>Welcome to CodeSandbox</h2>
        <select onChange={this.onChangeCategory}>
          <option value="">Select Category</option>
          {categories.map(({ id, name }) => (
            <option key={id} value={name}>
              {name}
            </option>
          ))}
        </select>
        {loadingLinks && <p>Loading links ...</p>}
        {hasCategoryLinks && <CategoryLink links={categoryLinks} />}
      </div>
    );
  }
}

App.propTypes = {
  categories: PropTypes.arrayOf(
    PropTypes.shape({
      id: PropTypes.number,
      name: PropTypes.string
    })
  ),
  selectedCategory: PropTypes.string,
  categoryLinks: PropTypes.array,
  onChangeCategory: PropTypes.func,
  fetchCategories: PropTypes.func
};

App.defaultProps = {
  onChangeCategory: emptyFunction,
  fetchCategories: emptyFunction
};

const mapStateToProps = ({
  categories: {
    categories,
    selected: selectedCategory,
    categoryLinks,
    loadingLinks
  }
}) => ({
  categories,
  selectedCategory,
  categoryLinks,
  loadingLinks
});

const mapDispatchToProps = dispatch => ({
  fetchCategories: () => {
    dispatch(Action(FETCH_CATEGORIES));
  },
  onChangeCategory: category => {
    if (category === "") {
      dispatch(Action(RESET_LINKS));
      return;
    }
    dispatch(Action(SELECT_CATEGORY, { category }));
    dispatch(Action(FETCH_LINKS, { category }));
  }
});

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(App);

```

### /components/category-link.js

```sh
import React from "react";
import PropTypes from "prop-types";

const CategoryLink = ({ links }) => {
  return (
    <ul>
      {links.map(({ Link, API, Description }, i) => (
        <li key={i}>
          <a href={Link}>{API}</a>
          :: {Description}
        </li>
      ))}
    </ul>
  );
};

CategoryLink.propTypes = {
  links: PropTypes.array.isRequired
};

export default CategoryLink;
```

### /reducers/categories.js

```sh
import {
  FETCH_CATEGORIES_SUCCEEDED,
  SELECT_CATEGORY,
  RESET_LINKS,
  FETCH_LINKS,
  FETCH_LINK_SUCCEEDED
} from "../Actions/types";

const initialState = {
  categories: [],
  selected: null,
  categoryLinks: [],
  loadingLinks: false
};

function categoryReducer(state = initialState, action) {
  switch (action.type) {
    case FETCH_CATEGORIES_SUCCEEDED: {
      return {
        ...state,
        categories: [...action.data]
      };
    }
    case SELECT_CATEGORY: {
      return {
        ...state,
        selected: action.data.category
      };
    }
    case RESET_LINKS: {
      return {
        ...state,
        loadingLinks: false,
        categoryLinks: []
      };
    }
    case FETCH_LINKS: {
      return {
        ...state,
        loadingLinks: true,
        categoryLinks: []
      };
    }
    case FETCH_LINK_SUCCEEDED: {
      return {
        ...state,
        loadingLinks: false,
        categoryLinks: [...action.data]
      };
    }
    default:
      return state;
  }
}

export default categoryReducer;
```

### /reducers/index.js

```sh
import { combineReducers } from "redux";
import categories from "./categories";

export default combineReducers({
  categories
});
``` 

### /sagas/index.js

```sh
import { put, call, takeEvery, takeLatest } from "redux-saga/effects";

import Api from "../Api";

export function* fetchCategories() {
  try {
    const response = yield call(Api.fetchCategories);
    const data = response.data.map((name, id) => ({ id, name }));
    yield put({ type: "FETCH_CATEGORIES_SUCCEEDED", data });
  } catch (error) {
    yield put({ type: "FETCH_CATEGORIES_FAILED", error });
  }
}

export function* fetchLinks(action) {
  try {
    const response = yield call(Api.fetchLinks, action.data.category);
    yield put({ type: "FETCH_LINK_SUCCEEDED", data: response.data.entries });
  } catch (error) {
    yield put({ type: "FETCH_LINK_FAILED", error });
  }
}

export default function* rootSaga() {
  yield takeEvery("FETCH_CATEGORIES", fetchCategories);
  yield takeLatest("FETCH_LINKS", fetchLinks);
}
``` 

### /utils/index.js

```sh
export const emptyFunction = () => {};
```

### Demo 
https://codesandbox.io/s/8xkmpm6k5l
