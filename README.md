# Udemy-Advanced-React-Redux
Udemy advanced react redux笔记
#### Middleware中间件

中间件是redux的高级功能，可以处理异步等情况。这个教程中其实并没有讲太多middleware的实现机理，这也是这个课程的一个问题，深度并不是很够（具体的源码层面的东西可去看网上的文章），但是从应用层面还是做了很多debug的，尽可能的去理解它的工作过程。

动手写三个middleware: redux-thunk, redux-promise和一个自定义middleware。

redux-promise:

```
export default function({ dispatch }) {
  return next => action => {
    // if action does not have payload or, the payload does not have a .then
    // property we dont care about it, send it on
    if (!action.payload || !action.payload.then) {
      return next(action);
    }

    // make sure the action's promise resolves
    action.payload.then(response => {
      const newAction = { ...action, payload: response };
      dispatch(newAction);
    });
  };
}
```
redux-promise的核心就是这样了，具体的代码实现上会有些不同。

redux-thunk

```
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

redux-thunk的核心其实是下面三行:
```
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
```

另外，还开发了一个custom middleware：stateValidate，它的定义是检查redux store中state的类型正确性。这个check的功能是通过JSON Schema实现的，所以整个middleware非常的简单，事实上，大部分middleware都很简短。

```
import tv4 from 'tv4';
import stateSchema from './stateSchema';

export default ({ dispatch, getState }) => next => action => {
	next(action);
	
	if (!tv4.validate(getState(), stateSchema)) {
		console.warn('Invalid state schema detected');
	}
};
```
这个middleware的开发有两个亮点：
1. 最外层接受的参数其实是一个包含了：dispatch和getState这两个方法的对象，这个特点可以从applyMiddleware的源码中得到确认；
2. 注意到在next(action)后面还有代码，next函数并不是一定要放在middleware的最后面。那next后面的代码是如何执行的呢？这里可以联想到Koa框架中middleware也有这样的使用。所以可以得出结论：Koa和Redux middleware都是洋葱模型，而Expressjs的middleware是线性模型。关于这一点可以看下面这篇文章：http://perkinzone.cn/2017/08/15/Redux,Koa,Express%E4%B9%8Bmiddleware%E6%9C%BA%E5%88%B6%E5%AF%B9%E6%AF%94/


#### High order component高阶组件

本项目中要开发一个requireAuth的高阶组件，它的逻辑是：当用户想进入某个react组件的时候，通过登录auth的状态来判断是否允许access这个组件。因为很多不同的组件都可能会有这个需求，所以自然就可以进行抽象的操作，而高阶组件就是进行抽象的一种方法。

这个项目还给出了进行高阶组件开发的四个标准步骤：
1. 首先在现有的一个组件中，实现相应的逻辑，比如这个例子中的判断auth状态并控制access的逻辑。
2. 新建一个高阶组件，构建它的基本架构。
3. 把刚刚的逻辑迁移到高阶组件中。
4. 从高阶组件向childComponent传递必须的各种属性（props等）

```
import React, { Component } from 'react';
import { connect } from 'react-redux';

// 高阶组件
export default (ChildComponent) => { 
	
	class ComposedComponent extends Component {
		// reuse的逻辑部分
		componentDidMount() {
			this.shouldNavigateAway();
		}
		componentDidUpdate() {
			this.shouldNavigateAway();
		}
		shouldNavigateAway() {
			if (!this.props.auth) {
				this.props.history.push('/')
			}
		}
		// render ChildComponent
		render() {
			return <ChildComponent {...this.props} />; // 传入props
		}
	}
	
	function mapStateToProps(state) {
		return { auth: state.auth };
	}
	
	return connect(mapStateToProps)(ComposedComponent);
}
```

#### 测试

在实际项目中完整的单元测试和高度自动化的CI/CD流程，都是工程化的必须。这个部分主要讲解的就是React/Redux项目进行单元测试的方法和技术。

Facebook自己出了一套进行单元测试的框架Jest，如果通过create-react-app脚手架创建项目的话 ，Jest这个库是内置的，不用另外install. 

单元测试文件的规范：
1. Files with .js suffix in __tests__ folders.
2. Files with .test.js suffix.
3. Files with .spec.js suffix.
上面这三类情况，只要是在src文件夹内，当运行npm test的时候，这些单元测试文件内的测试case就会运行。

组件基本功能的测试
```
import React from 'react';
import { mount } from 'enzyme';
import CommentBox from 'components/CommentBox';
import Root from 'Root';

let wrapped;

// beforeEach函数
beforeEach(() => {
 // enzyme的实例，enzyme提供了三个API，mount是其中的一个，也是功能支持最全面的一个API
  wrapped = mount(
    <Root>
      <CommentBox />
    </Root>
  );
});

// afterEach函数
afterEach(() => {
  wrapped.unmount();
});

// it函数
it('has a text area and 2 button', () => {
  //enzyme实例的find函数
  expect(wrapped.find('textarea').length).toEqual(1);
  expect(wrapped.find('button').length).toEqual(2);
});
```
此处使用了一个关键的框架enzyme，这是Airbnb出的针对react的测试框架。enzyme提供一组API，使用的风格和jQuery非常的类似，本身没什么难度，在需要的场景使用需要的API即可。

如果想使用enzyme库的话，需要在src文件夹内配置一个setupTests.js的文件,如下：
```
import Enzyme from 'enzyme';
import Adapter from 'enzyme-adapter-react-16';

Enzyme.configure({ adapter: new Adapter() });
```

除了基本功能之外，还有下面这些略微复杂的功能：
```
describe('the text area', () => {
  beforeEach(() => {
    // simulate模拟一个事件和事件的参数 
    wrapped.find('textarea').simulate('change', {
      target: { value: 'new comment' }
    })
    wrapped.update();
  });

  it('has a text area that users can type in', () => {
    expect(wrapped.find('textarea').prop('value')).toEqual('new comment');
  });

  it('when form is submitted, text area gets emptied', () => {
    wrapped.find('form').simulate('submit');
    wrapped.update();
    expect(wrapped.find('textarea').prop('value')).toEqual('');
  });
});
```

除了组件之外，redux的各个部分(action, reducer等)都可以测试。

对action的测试：

```
import { saveComment } from 'actions';
import { SAVE_COMMENT } from 'actions/types';

describe('saveComment', () => {
  it('has the correct type', () =>  {
    // 生成一个action实例
    const action = saveComment();
    // test action的type类型
    expect(action.type).toEqual(SAVE_COMMENT);
  });

  it('has  the correct payload', () => {
    const action = saveComment('New Comment');
    // test action的payload属性
    expect(action.payload).toEqual('New Comment');
  });
});
```

saveComment是一个action，定义如下：
```
export function saveComment(comment) {
  return {
    type: SAVE_COMMENT,
    payload: comment,
  };
}
```

对reducer的测试：

```
import commentsReducer from 'reducers/comments';
import { SAVE_COMMENT } from 'actions/types';

it('handles actions of type SAVE_COMMENT', () => {
  // 造一个action实例
  const action = {
    type: SAVE_COMMENT,
    payload: 'New Comment',
  };
  // 用reducer生成一个新的state	
  const newState = commentsReducer([], action);
  // 判断state的新状态是否正确
  expect(newState).toEqual(['New Comment']);
});

it('handles action with unknown type', () => {
  const newState = commentsReducer([], { type: 'ASDFASDFSDF' });
  expect(newState).toEqual([]);
})
```

commentsReducer是一个reducer定义如下：
```
export default function(state = [], action) {
  switch (action.type) {
    case SAVE_COMMENT:
      return [...state, action.payload];
    case FETCH_COMMENTS:
      const comments = action.payload.data.map(comment => comment.name);
      return [...state, ...comments];
    default:
      return state;
  }
}
```
对HTTP请求的测试

比如，下面这个action，会利用redux-promise来处理异步请求：
```
export function fetchComments() {
  const response = axios.get('http://jsonplaceholder.typicode.com/comments');

  return {
    type: FETCH_COMMENTS,
    payload: response,
  };
}
```

```
import React from 'react';
import { mount } from 'enzyme';
import moxios from 'moxios';
import Root from 'Root';
import App from 'components/App';


beforeEach(() => {
  // 利用moxios拦截axios的请求，并fake一个response回去
  moxios.install();
  moxios.stubRequest('http://jsonplaceholder.typicode.com/comments', {
    status: 200,
    response: [{name: 'Fetched #1'}, {name: 'Fetched #2'}]
  });
});

afterEach(() => {
  moxios.uninstall();
});

it('can fetch a list of comments and display them', (done) => { // done回调参数
  // enzyme实例
  const wrapped = mount(
    <Root>
      <App />
    </Root>
  );
  // 模拟click事件，发出fetchComments的异步action
  wrapped.find('.fetch-comments').simulate('click');
  
  //利用moxios的wait函数来处理拦截的异步效果
  moxios.wait(() => {
    wrapped.update();
    expect(wrapped.find('li').length).toEqual(2);
    done(); //执行done，通知jest框架测试结束
    wrapped.unmount();
  });


});
```

对于异步的测试算是比较复杂的情况了，它主要涉及下面几点问题：
1. 首先Jest是基于node的runner，它通过JSDOM来模拟浏览器的行为，但是它对于axios的http请求的支持不好。
2. 对于上面这个问题，可以引入moxios这个库，它会拦截axios的请求，然后返回faked的数据用来测试。
3. 另外，这种http请求都是异步，所以如果直接去expect结果是不行的，需要要让程序wait一段时间。可以用settimeout来简单模拟，当然也可以使用moxios提供的wait函数。
4. 另外，jest的每个it函数的第二个回调函数这个参数，都接受done这个回调。它的作用就是在这种异步的场合，只有手动调用了done这个回调，才表明测试结束，否则jest无法知道异步操作合适完成。
