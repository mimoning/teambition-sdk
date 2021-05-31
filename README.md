[![CircleCI](https://circleci.com/gh/teambition/teambition-sdk/tree/release.svg?style=svg)](https://circleci.com/gh/teambition/teambition-sdk/tree/release)
[![Coverage Status](https://coveralls.io/repos/github/teambition/teambition-sdk/badge.svg?branch=release)](https://coveralls.io/github/teambition/teambition-sdk?branch=release)
[![Dependency Status](https://david-dm.org/teambition/teambition-sdk.svg)](https://david-dm.org/teambition/teambition-sdk)
[![devDependency Status](https://david-dm.org/teambition/teambition-sdk/dev-status.svg)](https://david-dm.org/teambition/teambition-sdk?type=dev)

# isomorphic-sdk for Teambition APIs

[![Greenkeeper badge](https://badges.greenkeeper.io/teambition/teambition-sdk.svg)](https://greenkeeper.io/)

## 安装

```bash
npm install teambition-sdk --save-dev
```
如果要求在不支持 Fetch API 的浏览器运行，请安装 polyfill 如 [whatwg-fetch](https://github.com/github/fetch)。

## 开发

目前主分支是 `release`，而 `master` 只应用于部分处于维护状态的老项目。常用命令如：

```bash
yarn               # 安装依赖
npm run build_test # 构建测试
npm test           # 跑一遍完整测试
npm watch          # 在开发时，以监听模式跑测试，每次代码修改都会重跑测试，帮助及时发现问题
```

## 发布
注意目前发布要求使用 NPM 二步验证，请参考[文档](https://docs.npmjs.com/configuring-two-factor-authentication#sending-a-one-time-password-from-the-command-line)，在发布命令后面跟随 `--otp=验证码`。
```bash
# only publish sdk
npm run preversion   # 查询 npm 官方库，获取当前发布的最新正式版本和最高的预发版本号，避免打版本时出错
npm version v0.12.68 # 打正式版本 0.12.68（包括创建相应标签）
npm version v0.12.69-alpha.0-readme # 打预发版本 0.12.69-alpha.0-readme（包括创建相应标签）
npm run publish_sdk  # 在本地 build 当前代码并发布到 npm 官方库
# 完成发布后，推荐将相应标签（tag）推到远端，如 `git push origin v0.12.69-alpha.0-readme`

# publish sdk, mock and socket
npm version xxx
npm run publish_all
```

## 设计理念

SDK 主要解决的是数据同步的问题。通俗点讲，就是在前端使用数据模型模拟出数据库的增删改查等操作。

为什么会有这种需求? 以 `https://api.teambition.com/tasks/:_id` 为例， Teambition 的 API 会返回下面格式的数据:

```json
{
  "_id": "001",
  "name": "task",
  "executor": {
    "_id": "002",
    "name": "executor 1",
    "avatarUrl": "https://xxx"
  },
  "subtasks": [
    {
      "_id": "003",
      "name": "subtask",
      "executor": {
        "_id": "004",
        "name": "executor 2",
        "avatarUrl": "https://xxx"
      }
    }
  ]
}
```

而倘若这个任务中包含的子对象，比如 `executor` 字段对应的数据通过其它 API 进行了变更:

```ts
/**
 * @url https://api.teambition.com/subtasks/:_id
 * @method put
 * @body {name: 'executor test'}
 */
SubtasksAPI.update('002', {
  name: 'subtask update'
})
  .subscribe()
```

在前端，需要自行处理与此 subtask 相关的所有变更情况。例如:

1. 包含这个子任务的列表中的这个子任务名字的变更。
2. 包含这个子任务的任务的详情页中，该子任务名字的变更。

然而在现有的 Teambition 数据模型中，需要在每一个 `Model` 或者 `Collection` 或者 `View` 中手动监听与自己相关联的数据，例如:

```js
// 匹配第一种情况
class MyTasksView extends Backbone.View {
  ...
  listen() {
    this.listenTo(warehouse, ':change:task', model => {
      // handler
    })
    this.listenTo(warehouse, ':change:subtask', model => {
      // handler
    })
  }
}
```

```js
// 匹配第二种情况

class SubtaskCollection extends Backbone.Collection {
  ...

  constructor() {
    this.on('add destroy remove change:isDone', () =>
      Socket.trigger(`:change:task/${this._boundToObjectId}`, {
        subtaskCount: {
          total: this.length
          done: this.getDoneSubTaskCount()
        }
      })
    )
  }
  getDoneSubTaskCount() {
    this.where({isDone: true}).length
  }
}

class TaskView extends Backbone.View {
  ...
  listen() {
    this.listenTo(this.taskModel, 'change', this.render)
  }
}
```

而在当前的设计中，所有的这种变更情况都在数据层处理，视图/业务 层只需要订阅一个数据源，这个数据源随后的所有变更都会通知到订阅者。
比如获取一个任务:

```ts
import 'rxjs/add/operator/distinctUntilKeyChanged'
import 'teambition-sdk/apis/task'
import { SDK } from 'teambition-sdk/SDK'
import { Component, Input } from '@angular/core'

@Component({
  selector: 'task-detail',
  template: `
    <div> {{ task$?.name | async }} </div>
    <div> {{ subtaskCount$ | async }} </div>
  `
})
export default class TaskView {

  @Input('taskId') taskId: string

  private task$ = this.SDK.getTask(this.taskId)
    .publishReplay(1)
    .refCount()

  private subtaskCount$ = this.task$
    .distinctUntilKeyChanged('subtasks')
    .map(task => ({
      total: task.subtasks.length,
      done: task.subtasks.filter(x => x.isDone).length
    }))
}
```


如果更加纯粹的使用 RxJS，甚至可以组合多种数据和业务:


```ts
import 'rxjs/add/operator/distinctUntilKeyChanged'
import 'rxjs/add/operator/distinctUntilChanged'
import 'teambition-sdk/apis/permission'
import 'teambition-sdk/apis/task'
import 'teambition-sdk/apis/project'
import { SDK } from 'teambition-sdk'
import { Component, Input } from '@angular/core'
import * as moment from 'moment'
import { errorHandler } from '../errorHandler'

@Component({
  selector: 'task-detail',
  template: `
    <div [ngClass]="{'active': permission$.canEdit | async}"></div>
    <div> {{ task$?.name | async }} </div>
    <div> {{ subtaskCount$ | async }} </div>
    <div> {{ dueDate$ | async }} </div>
  `
})
export default class TaskView {

  @Input('taskId') taskId: string

  private task$ = SDK.getTask(this.taskId)
    .catch(err => errorHandler(err))
    .publishReplay(1)
    .refCount()

  private subtaskCount$ = this.task$
    .distinctUntilKeyChanged('subtasks')
    .map(task => ({
      total: task.subtasks.length,
      done: task.subtasks.filter(x => x.isDone).length
    }))

  private dueDate$ = this.task$
    .map(task => moment(task.dueDate).format())

  private project$ = this.task$
    .distinctUntilKeyChanged('_projectId')
    .switchMap(task => SDK.getProject(task._projectId))
    .catch(err => errorHandler(err))
    .publishReplay(1)
    .refCount()

  private permission$ = this.task$
    .distinctUntilChanged((before, after) => {
      return before._executorId === after._executorId &&
        before._projectId === after._projectId
    })
    .switchMap(task => {
      return this.project$
        .distinctUntilKeyChanged('_defaultRoleId')
        .switchMap(project => {
          return SDK.getPermission(task, project)
        })
    })
    .catch(err => errorHandler(err))
    .publishReplay(1)
    .refCount()
```

在这种场景下，关于 task 的任何变更 (tasklist 变更，executor 变更，stage 变更等等，权限变化) 都能让相关的数据自动更新，从而简化 View 层的逻辑。
