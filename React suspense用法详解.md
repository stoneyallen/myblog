
React.suspense是大家用的比较少的功能，它早在2018年的16.6.0版本中就已发布。它的相关用法有些已经比较成熟，有的相对不太稳定，甚至经历了重命名、删除。

下面一起来了解下它的主要用法、场景。

#### 1.suspense配合lazy实现code spliting

import是webpack中的一种code spliting的用法，但是import的文件返回的是一个promise，必须封装之后才能使用，例如react-loadable的封装方法

    function Loadable(opts) {
      const { loading: LoadingComponent, loader } = opts
      return class LoadableComponent extends React.Component {
        constructor(props) {
          super(props)
          this.state = {
            loading: true, // 是否加载中
            loaded: null  // 待加载的模块
          }
        }
        componentDidMount() {
          loader()
            .then((loaded) => {
              this.setState({
                loading: false,
                loaded
              })
            })
            .catch(() => {})
        }
    
        render() {
          const { loading, loaded } = this.state
          if (loading) {
            return <LoadingComponent />
          } else if (loaded) {
            // 默认加载default组件
            const LoadedComponent = loaded.__esModule ? loaded.default : loaded;
            return <LoadedComponent {...this.props}/>
          } else {
            return null;
          }
        }
      }
    }

在promise返回后更新组件，如果使用suspense改写react-loadable，将会更加优雅

    const ProfilePage = React.lazy(() =>  import('./ProfilePage'));
    
    
    <Suspense fallback={<Spinner />}>
    
      <ProfilePage />
    
    </Suspense>

#### 2.1 请求数据时解决loading问题

    let status = "pending";
    let result;
    const data = new Promise(resolve => setTimeout(() => resolve("结果"), 1000));
    
    function wrapPromise(promise) {
      let suspender = promise.then(
        r => {
          status = "success";
          result = r;
        },
        e => {
          status = "error";
          result = e;
        }
      );
      if (status === "pending") {
         throw suspender;
      } else if (status === "error") {
        throw result;
      } else if (status === "success") {
        return result;
      }
    }
    
    
    function App(){
        const state = wrapPromise(data);
        
      return (<div>{state}</div>);
    }
    
    function Loading(){
        return <div>..loading</div>
    }
    
    class TodoApp extends React.Component {
      
      render() {
        return (
          <React.Suspense fallback={<Loading></Loading>}> 
            <App />
          </React.Suspense>
        )
      }
    }
    
    ReactDOM.render(<TodoApp />, document.querySelector("#app"))
    

[源码在此](https://jsfiddle.net/shijian/7a0whmu5/37/%29)  
上面的写法比较奇怪，在组件App中请求数据state时，一开始返回throw promise，这是为了让suspense捕捉到error，返回loading组件，以上写法跟suspense的实现方式有关

    class Suspense extends React.Component { 
        state = { promise: null } 
        componentDidCatch(e) { 
            if (e instanceof Promise) { 
                this.setState(
                { promise: e }, () => { 
                    e.then(() => { 
                        this.setState({ promise: null }) 
                    }) 
                }) 
            } 
        } 
        render() { 
            const { fallback, children } = this.props 
            const { promise } = this.state 
            return <> 
                { promise ? fallback : children } 
            </> 
        } 
    }
    

从suspense源码可以看出，suspense捕捉到error后，会对其监听，当返回值时将loading改为children中的组件。  
但这时又会触发一次组件渲染，所以需要对请求结果缓存，最终变成上面的写法。  
这里有个官方例子可供参考，[传送门](https://codesandbox.io/s/frosty-hermann-bztrp)

#### 2.2 使用react-cache缓存

上面的例子非常反人类，在实际项目中基本不可能这样写，配合react-cache将会优雅许多

    import React, { Suspense } from "react";
    import { unstable_createResource as createResource } from "react-cache";
    
    const mockApi = () => {
      return new Promise((resolve, reject) => {
        setTimeout(() => resolve("Hello"), 1000);
      });
    };
    
    const resource = createResource(mockApi);
    
    const Greeting = () => {
      const result = resource.read();
    
      return <div>{result} world</div>;
    };
    
    const SuspenseDemo = () => {
      return (
        <Suspense fallback={<div>loading...</div>}>
          <Greeting />
        </Suspense>
      );
    };
    
    export default SuspenseDemo;

> **react-cache官方目前不推荐使用在线上项目中**

#### 3.配合ConcurrentMode解决loading的闪现问题

loading的闪现问题主要是因为api接口时间短，loading不该出现，需要对接口速度进行判断

不考虑suspense按照通常的写法，可以这么实现

    const timeout = ms => new Promise((_, r) => setTimeout(r, ms));
    
    const rq = (api, ms, resolve, reject) => async (...args) => {
      const request = api(...args);
      Promise.race([request, timeout(ms)]).then(resolve, err => {
        reject(err);
        return request.then(resolve);
      });
    };

suspense为我们提供了maxDuration属性，用来控制loading的触发时间

    import React from "react";
    import ReactDOM from "react-dom";
    const {
      unstable_ConcurrentMode: ConcurrentMode,
      Suspense,
    } = React;
    const { unstable_createRoot: createRoot } = ReactDOM;
    
    let status = "pending";
    let result;
    const data = new Promise(resolve => setTimeout(() => resolve("结果"), 3000));
    
    function wrapPromise(promise) {
      let suspender = promise.then(
        r => {
          status = "success";
          result = r;
        },
        e => {
          status = "error";
          result = e;
        }
      );
      if (status === "pending") {
         throw suspender;
      } else if (status === "error") {
        throw result;
      } else if (status === "success") {
        return result;
      }
    }
    
    
    function Test(){
        const state = wrapPromise(data);
        
      return (<div>{state}</div>);
    }
    
    function Loading(){
        return <div>..loading</div>
    }
    
    class TodoApp extends React.Component {
      
      render() {
        return (
          <Suspense fallback={<Loading></Loading>} maxDuration={500}> 
            <Test />
          </Suspense>
        )
      }
    }
    
    const rootElement = document.getElementById("root");
    
    createRoot(rootElement).render(
      <ConcurrentMode>
        <TodoApp />
      </ConcurrentMode>
    );
    

[源码地址在此](https://codesandbox.io/s/relaxed-architecture-tirzo)

**上面例子使用的是16.8.0版本**

例子中用到了**unstable_ConcurrentMode**、**unstable_createRoot**语法，unstable\_createRoot在16.11.0中已更名为createRoot，unstable\_ConcurrentMode在16.9.0中更名为unstable_createRoot  
在最新16.13.1中测试发现ReactDOM.createRoot并不存在，所以本例子只在**16.8.0**中测试

#### 总结

以上就是关于suspense的所有场景，目前api善不稳定，谨慎使用

#### 招聘

最近字节跳动前端急招，有感兴趣的请私信我，或者投递我邮箱**574745389@qq.com**  
前端base上海、北京、南京、深圳、杭州，岗位要求可参考[https://job.toutiao.com/s/7wokvh](https://job.toutiao.com/s/7wokvh)  
除了**前端**其他岗位的也欢迎投递

