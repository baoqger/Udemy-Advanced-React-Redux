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

