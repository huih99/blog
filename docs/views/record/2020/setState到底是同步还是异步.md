---
title:  setState什么时候是异步什么时候是同步
date: 2020-02-20
tags:
  - js
categories:
  - 笔记
---

# setState什么时候是异步什么时候是同步

1. 下列代码，最终打印的是什么
```js
    class A extends React.Component {
        constructor(props) {
            super(props);
            this.state = {
                num:0
            }
        }
        componentDidMount() {
            this.setState({
                num: this.state.num + 1
            })
            console.log(this.state.num)
            this.setState({
                num: this.state.num + 1
            })
            console.log(this.state.num)
            setTimeout(()=>{
                // 在setTimeout中不会触发setState的队列更新机制，所以每次的setState都会立即引发组件更新，更改state为最新值
                // 1.num = 1;
                this.setState({
                    num: this.state.num + 1
                })
                // 2. num = 2;
                console.log(this.state.num)
                this.setState({
                    num: this.state.num + 1
                })
                // 3. num = 3
                console.log(this.state.num)
            })
        }
    }
```
- 正确的答案是 0,0,2,3。

> 在react的更新机制中，存在着batchUpdate的机制,在通常情况下setState并不会引发组件立即更新，所以也就不能马上拿到改变后的state的值，所以前两次打印的值还是0。 这里的通常情况指的是**1.由React引发的事件处理，比如使用React事件机制绑定的各种事件(如onClick);2.React生命周期中触发的setState;**,除此之外的其他情况不会触发setState的队列更新机制，这时候的setState表现为同步机制，会立即更新state的值，引发一次更新。（如在setTimeout/setInterval,promise.then中使用setState,或者使用原生的事件绑定机制(如：el.addEventListener)绑定的事件中使用setState

1. setState中使用函数的情况
   ```js
    class B extends React.Component {
        constructor(props) {
            super(props);
            this.state = {
                num: 0
            }
        }
        componentDidMount() {
            this.setState((prevState,props) => {
                return {
                    num: prevState.num + 1
                }
            })
            this.setState((prevState,props) => {
                return {
                    num: prevState.num + 1
                }
            })
            this.setState((prevState,props) => {
                return {
                    num: prevState.num + 1
                }
            },() => {
                console.log(this.state.num) // 打印出来： num = 3
            })
        }
    }
   ```
在setState中使用函数的情况，最后num的值会变为3。函数接收的第一个参数：prevState表示上一次调用setState的值，这里经过了3次setState，每一次拿到的都是修改后的num值（这里注意：即便是使用函数方式设置state,实际组件中的state依然没有立即改变，这里的state更新机制依然跟上述第一种情况分析的更新机制相同，这里只是react处理使用setState中传入函数情况时，会把每次setState修改的结果作为参数传入下一个使用函数入参的setState中）。