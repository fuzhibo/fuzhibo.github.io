---
layout: post
title:  "构建自己的 VS Code Extension 的实践总结（一）"
date:   2019-12-25 13:10:29 +0800
categories: vscode,extension
---
&emsp;&emsp;这次实践的目的主要是完成如下几个特性：
* 能够让插件可以在 vs code 的 setting 页面中进行配置，目前可以只支持 GitLab URL 和 Token 的配置。
* 增加一个 precheck 的命令，可以对配置的 GitLab URL 和 Token 进行校验。

#### 新增自定义的命令映射
&emsp;&emsp;在基本的开发环境搭建好后，现在至少已经可以启动调试并通过控制台命令触发一个 `Hello World` 的消息通知。那么在这个基础上修改一个自己的命令并不难。直接修改 `package.json` 下的 `contributes.commands` 部分增加一个自己定义的命令 `gldevops.precheck`。
```json
{
    /*.......*/
    "contributes": {
		"commands": [
			{
				"command": "gldevops.precheck",
				"title": "GitLab DevOps Management: Begin to do precheck."
			}
        ]
        /*.......*/
    }
    /*.......*/
}
```
&emsp;&emsp;同时修改的还有 `activationEvents` 列表。
```json
{
    /*.......*/
    "activationEvents": [
        "onCommand:gldevops.precheck"
        /*.......*/
    ]
    /*.......*/
}
```
&emsp;&emsp;配置修改完成后，对应到代码层面上的更改主要是使用 `vscode.commands.registerCommand` 方法注册指定的函数为 `gldevops.precheck`。这样才能建立上映射。
```typescript
let disposable = vscode.commands.registerCommand('gldevops.precheck', () => {/*Your code here*/});
context.subscriptions.push(disposable);
```
#### 从 vs code 的 setting 页面中读取自己的配置
&emsp;&emsp;首先，还是从 `package.json` 入手，通过配置可以让 vs code 自动识别并生成相关的配置选项到 setting 页面之中。
```json
{
    /*.......*/
    "configuration": {
        "title": "GitLab DevOps Management",
        "properties": {
            "gitlabDevopsMgt.instanceUrl": {
                "type": "string",
                "default": "https://gitlab.com",
                "description": "Your GitLab instance URL (default is https://gitlab.com)"
            },
            "gitlabDevopsMgt.personalAccessToken": {
                "type": "string",
                "default": null,
                "description": "Your GitLab Personal Access Token"
            }
        }
    }
    /*.......*/
}
```
&emsp;&emsp;在代码中获取相关的配置，并使用。
```typescript
let gitlabUrl = vscode.workspace.getConfiguration('gitlabDevopsMgt').instanceUrl.match('http[s]*://[^/]+');
let gitlabToken = vscode.workspace.getConfiguration('gitlabDevopsMgt').personalAccessToken;
```
#### 校验填写的 GitLab URL 和 Token
&emsp;&emsp;获取到指定的 URL 和 Token 后，需要对它们进行校验。校验方法这里使用获取 GitLab 服务器版本信息的办法，通过 `/api/v4/version` 来获取到版本信息进行验证。  
&emsp;  
&emsp;&emsp;这里需要用到一个 HTTP 请求连接的库，这里用 needle 来作为 backend。在 `package.json` 中需要增加对 `needle` 库的引用。
```json
"devDependencies": {
    /*.......*/
    "@types/needle": "^2.4.0",
    /*.......*/
    "needle": "^2.4.0"
    /*.......*/
}
```
&emsp;&emsp;在代码层面，实现一个模块封装 HTTP 相关操作。
```typescript
import * as needle from 'needle';

// The module 'requests' implementation a HTTP request operator class
export namespace HttpClient {
    export class Requests {

        constructor() {
            console.log("Requests initialize...");
        }

        // do some implementation
        public get(url: string, options: any = undefined) {
            console.log("Begin to do GET request for " + url + " ...");
            var resp = needle('get', url, options);
            return resp;
        }

        public post() {
            console.log("Begin to do POST request...");
        }

        public put() {
            console.log("Begin to do PUT request...");
        }

        public delete() {
            console.log("Begin to do DELETE request...");
        }
    }
}
```
&emsp;&emsp;对于自己开发的模块，可以增加一个单元测试，这样可以有效的提高模块的稳定性。
```typescript
import * as vscode from 'vscode';

import * as assert from 'assert';

import * as requests from '../../requests';

suite('Requests Class Test Suite', () => {
    vscode.window.showInformationMessage('Start Requests test...');

    test('GET request test', () => {
        // do get request test
        var options = {
            open_timeout: 20000,
            headers: {
                'Content-Type': 'application/json',
                'PRIVATE-TOKEN': 'AXUiDDXJrK19HfqypV7x'
            }
        };
        var req = new requests.HttpClient.Requests;
        var res = req.get('https://gitlab.com/api/v4/version', options);
        assert.notEqual(res, undefined);
        res.then((resp) => {
            assert.equal(resp.statusCode, 200);
            assert.notEqual(resp.body, null);
            assert.notEqual(resp.body, undefined);
            assert.notEqual(resp.body.version, undefined);
            assert.notEqual(resp.body.revision, undefined);
            console.log("Status code: " + resp.statusCode);
            console.log("GitLab version: " + resp.body.version);
            console.log("GitLab revision: " + resp.body.revision);
        });
        res.catch((err) => {
            assert.equal(err, undefined);
            console.log("Get Error: " + err);
        });
    });

    test('POST request test', () => {
        // do post request test
        var req = new requests.HttpClient.Requests;
        req.post();
    });

    test('PUT request test', () => {
        // do put request test
        var req = new requests.HttpClient.Requests;
        req.put();
    });

    test('DELETE request test', () => {
        // do delete request test
        var req = new requests.HttpClient.Requests;
        req.delete();
    });
});
```
#### 总结
&emsp;&emsp;这一阶段的代码可以看[这里](https://github.com/fuzhibo/gitlab-devops-management/tree/lession1)。这个阶段，已经完成了 vs code extension 本身的搭建和对接 GitLab API 的初步实现。那么下一阶段，需要完成的任务有：
* 在 vs code 的左侧边栏上增加一个属于这个插件自己的 Logo，当用户点击这个 Logo 的时候进行 GitLab 的初始化校验并弹出一个工作面板。
* 对这个 Token 所能够显示的 Group 和 Project 用 Tree List 空间进行展示（最好和 vs code 风格统一），并标明权限。