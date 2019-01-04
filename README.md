# このレポジトリについて
reduxの勉強のために作成。コードは[Udemyのreduxコース](https://www.udemy.com/react-redux/learn/v4/t/lecture/12586846?start=68)を写経した成果物です。

学んだことの備忘録をあとで見返せるようにまとめておきます。

# 環境構築について
npx create-react-app blogで空のプロジェクトを作成

必要なパッケージをインストール
yarn add react-redux axios redux-thunk

src以下のファイル削除して順次必要なファイル群を生成。


# Reduxの基本
1. ディレクトリ
actions, components, reducers, containers, apis, constantsなど。

2. Reducer作成
bookReducer.jsのように命名すると検索しやすい。functional componentを作成してexportする。
reducers/index.jsなどでimportし、combineReducersの中に`books: BooksReducers`と記述することで、booksという名前の、アプリケーション全体からアクセス可能なstateが作られる。

3. Container作成
まず、react-reduxからconnect関数をimportする。次に、通常のReactよろしくクラスコンポーネントを定義。mapStateToProps関数を下のように定義する。

```js
mapStateToProps(state) {
  return {
    asdf: '123',
    books: state.books,
  }
}
```

ここでreturnされるオブジェクトはクラス内で`this.props.asdf`などとしてpropsからアクセス可能になる。引数のstateはreducerで定義したもの。
最後に`export default connect(mapStateToProps)(BookList)`としてヒモ付が完了。

4. ComponentからのContainer呼び出し
アプリケーション起動直後はstateにはnullが渡るので、`this.props.hoge`で拾った値がnullとなるため、`hoge.fuga`とするとエラーとなる。
これを防ぐため、最初に`if(this.props.hoge) { return <div>loading</div>}`のようなロード中であることを示すdiv要素を返すようにするのがセオリー。

5. Action Creator作成
actions/index.jsなどにアクション用の関数を定義してexportする。戻り値は基本的にはtypeとpayloadをもつオブジェクト。関数を返したい場合などはredux-thunkを導入。この関数をaction-creatorと呼ぶ。返されるオブジェクトをaction（オブジェクト）と呼ぶ。

6. ActionとContainerをつなぐ
actionを実行するcontainerからaction.jsをimportし、reduxのbindActionCreatorsもimportする。
次に、以下のようにmapDispatchToProps関数を定義する。

```js
function mapDispatchToProps(dispatch) {
  return bindActionCreators({ selectBook: selectBook }, dispatch)
}
```

これにより、selectBookというアクションが呼ばれる度にreducerにその戻り値が渡される。
bindActionCreators内のオブジェクトはpropsとしてcontainerに渡される。

connect関数の第2引数に`mapDispatchToProps`を追加することで紐づけは完了。

7. providerとstoreの準備
エントリファイルとなるindex.jsなどで以下のようにstoreを作成し、reducerとのヒモ付を行う。また、middlewareとのヒモ付けを行う。

```js
import { Provider } from 'react-redux';
import { createStore } from 'redux';

import App from './components/app';
import reducers from './reducers';

const store = createStore(reducers);

ReactDOM.render(
  <Provider store={store} >
    <App />
  </Provider>
  , document.querySelector('.container'));
```

上記までで基本的なreact-reduxの実装はできそう。


# APIの実装
次はAPIの実装についてメモ。react/reduxでAPI叩きたい場合は面倒なことにredux-thunkを使う必要がある。（正確にはmiddlewareが必要。)

## 手順
reduxのアクションでrequestを発行する場合はredux-thunkが必要なので予めインストールする。

まず、typeを持つオブジェクトを返すaction creatorを作成する。
データをロードしたいコンポーネントで読み込み、componentDidMount時に`this.props.fetchPosts()`のように呼び出す。

propsで呼び出せるように、connectメソッドをreact-reduxから読み込んでおき、ファイルの最後に`export default connect(null, {fetchPosts})(PostList);`のように記述する。stateをmapしない場合はconnectの第一引数にnullを指定する。また、第2引数はes6の文法で{fetchPosts: fetchPosts}の省略形で記述できる。

次に、actionをredux-thunkを使って非同期化する。具体的にはdispatchとgetState関数を引数に持つ関数を返すようにactionを書き換える。こうすることで、APIリクエストが終わるまでdispatch実行を待機してくれる(らしい)。

なお、apiの処理はapisフォルダなどを作成してそちらに分けるのがわかり良さそう。

アクションの非同期化の例は以下の通り。アロー関数使ってきれいにまとめてある。

```js
import jsonPlaceholder from '../apis/jsonPlaceholder'

export const fetchPosts = () => async dispatch => {
  const response = await jsonPlaceholder.get('/posts')

  dispatch({
    type: 'FETCH_POSTS',
    payload: response,
  })
};
```

なお、thunkを使うにはストア作成時にthunkを組み込むようにする。

```js
import React from 'react';
import ReactDOM from 'react-dom';
import { Provider } from 'react-redux';
import {
  createStore,
  applyMiddleware
} from 'redux';
import thunk from 'redux-thunk';

import App from './components/App';
import reducers from './reducers';

const store = createStore(reducers, applyMiddleware(thunk));

ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>
  , document.querySelector('#root'));
```


# Reducerの基本ルール
1. undfined以外を返すようにする
2. reducerは直近のstateとアクションに基づいて次のstateやデータを作成する。

一度目の呼び出しでstateにはundefinedが入っている。が、このままではreducer側でエラーとなる恐れがあるのでデフォルト値にnullを入れるようにする。
Reducerが2回目移行に呼ばれると直前のstateが渡される。

3. reducerの中でAPIリクエストや新しくデータを取得する処理を呼んではならない。
->あくまで引数となるstateとアクションに基づく「処理」のみ許容すること。
->pure reducerと呼ぶっぽい。

4. 引数として渡すstateに変更を加えてはならない。
->javascriptではarrayやobjectは簡単に変更できてしまうので注意が必要。
（一方でstringや数値はimmutable）


# その他のメモ
## dumbコンポーネント
reduxにアクセスしないコンポーネントのこと。逆に、reduxからデータを受け取るコンポーネントはcontainerと呼んで区別することが多い。ディレクトリを分ける場合はcomponentsとcontainersのように切れば良さそう。

## ReactとReduxの関係
ReactとReduxは異なる別々のライブラリ。これらをつないで使いたい場合はreact-reduxが必要。

## アクションの伝搬
アクションは全reducerに送られる。

## Reducerについて

undefinedのstateをactionで返すことはできないのでdefaultでstate=nullとする。
```js
export const activeBookReducer = (state = null, action) => {
  switch(action.type){
  case 'BOOK_SELECTED':
    return action.payload
  }

  return state
}
```

モジュールのimport/exportは{}をつける/つけないの基準がわかりにくい。そのため、予め統一させておいたほうが楽ができそう。
基本的には`const hoge = ...`と定義して、`export default hoge`とexportして、`import hoge from '../hoge'`などと{}なしに統一するとかで良いのでは？

なお、connectやrootReducerは
`export default rootReducer`としてexportしないとエラーが出たりするので注意。

## middlewareについて
middlewareはdispatchの直前に実行される。


## JsonPlaceholderについて
REST APIの実験をするのに使える外部APIサービス
https://jsonplaceholder.typicode.com/
