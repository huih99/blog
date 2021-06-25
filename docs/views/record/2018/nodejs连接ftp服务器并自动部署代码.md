---
title: 使用nodejs将代码自动部署到测试服务器
date: 2018-11-22
tags:
  - js
  - nodejs
categories:
  - 笔记
---

>能够自动化处理的坚决不手动
<!-- more -->

## 想法的产生

我司的项目在需要改动或者bug修复后，先经由开发人员自己测试无问题，然后上传至测试服务器由测试人员进行功能测试。上传至测试服务器这一步是怎么做的呢，首先打开ftp软件，然后登陆，连接测试服务器，然后选择项目打包目录，选择所有文件上传至服务器。而且需要先删除测试服务器文件，因为每次打包文件带hash，所以不会同名覆盖，造成存在多个不同版本的项目文件。步骤很烦琐，而且有时需要频繁上传测试服务器给测试人员，终于有一天，我不想忍受这种方式了，而且刚好近段时间任务没有很忙，所以就研究着怎么通过npm脚本直接就给上传了。

## 实践
我其实一开始是想找有没有vscode的ftp/sftp插件，直接一键上传，可惜的是找了很久并没有符合要求的扩展。

后来就想直接通过npm命令直接就上传了，所以想法是写一个pushServer.js，直接node pushServer.js就完事了。然后百度了下通过nodejs如何连接ftp/sftp，在npm库里有很多关于sftp/ftp的包，我找了一个下载量不错然后api也比较简洁的包**ssh2**。直接npm i ssh2 -D就好了。然后本地文件的处理主要是使用nodejs自带的**fs**模块来进行处理。

## 方案

本次要实现的这个上传服务器的功能包含上传、备份、上传失败后自动回滚这三个功能。

下面是代码实现：

```js
//pushServer.js

const Client = require('ssh2').Client;
const path = require('path');
const fs = require('fs');

//服务器配置，包括ip，端口，用户名和密码
const serverConfig = require('../devServer.config.json');

// 需要上传到的远程服务器的文件夹路径
const remotePath = '/home/zrx_dev/docker/02nginx/own/build/dl/merchant';
//本地打包代码的路径
const localPath = path.join(__dirname, '..', 'dist');
const FIX_FILENAME = '_backup';
/**
 * 将err-callback类型的异步函数转为Promise
 *
 * @param {*} fn 需要转换的函数
 * @param {*} context 修正函数的this指向
 */
const convertPromise = (fn, context) => (...args) =>
  new Promise((resolve, reject) => {
    fn.apply(context || this, [
      ...args,
      (err, data) => {
        if (err) {
          reject(err);
        } else {
          resolve(data);
        }
      }
    ]);
  });

// fs模块中需要用到的两个函数转为promise
const promiseFs = {
  stat: convertPromise(fs.stat, fs), //检查文件路径是单文件还是文件夹
  readdir: convertPromise(fs.readdir, fs) //读取文件目录下所有文件
};

//检查远程服务器中是否存在该路径
function checkPathInFtp(sftp, targetPath) {
  return new Promise((resolve, reject) => {
    sftp.readdir(targetPath, err => {
      if (err) {
        reject(err);
      } else {
        resolve();
      }
    });
  });
}

//读取服务器指定路径下所有文件
function readdirInFtp(sftp, dir) {
  return new Promise((resolve, reject) => {
    sftp.readdir(dir, (err, files) => {
      if (err) {
        reject(err);
      } else {
        resolve(files);
      }
    });
  });
}

//删除服务器下指定路径的文件
function deleteFileInFtp(sftp, filePath) {
  const fStat = convertPromise(sftp.stat, sftp); //检查路径是单文件还是文件夹
  const fUnlink = convertPromise(sftp.unlink, sftp); //删除单文件的方法
  const fRmdir = convertPromise(sftp.rmdir, sftp); //删除目录

  //删除文件夹时是不能直接删除的，必须将文件夹下所有子文件都删除完后才能删除目录,所以需要遍历目录下所有文件使用递归操作删除所有文件
  return new Promise(async (resolve, reject) => {
    try {
      const stats = await fStat(filePath);
      const isDirectory = stats.isDirectory();
      if (isDirectory) {
        // 如果该路径是文件夹，则读取所有文件
        const files = await readdirInFtp(sftp, filePath);
        for (let file of files) {
          await deleteFileInFtp(sftp, `${filePath}/${file.filename}`);
        }
        //待所有子文件及子文件夹都删除完成后，移除此目录
        await fRmdir(filePath);
      } else {
        //直接删除文件
        await fUnlink(filePath);
      }
    } catch (e) {
      console.log(`删除目录失败：${e}`);
    } finally {
      resolve();
    }
  });
}

// 备份服务器文件
function backup(sftp, filePath) {
  // 通过修改文件名完成备份，如果上传服务器失败，则删除上传文件并恢复此备份
  return new Promise(resolve => {
    sftp.rename(filePath, filePath + FIX_FILENAME, err => {
      if (err) {
        console.log(`备份失败：${err}`);
      } else {
        console.log('备份成功');
      }
      resolve();
    });
  });
}

//恢复上一次备份的文件内容
function recovery(sftp, filePath) {
  return new Promise(resolve => {
    sftp.rename(filePath + FIX_FILENAME, filePath, err => {
      if (err) {
        reject(`重命名目录失败：${err}`);
      } else {
        console.log('恢复上一版本成功');
        resolve();
      }
    });
  });
}

//版本回滚
async function rollback(sftp, filePath) {
  try {
    // 先删除本次上传的文件内容
    await deleteFileInFtp(sftp, filePath);
    // 恢复上一次备份内容
    await recovery(sftp, filePath);
  } catch (e) {
    console.log(`回滚失败：${e}`);
  }
}

// 文件上传
class UpFile {
  constructor(sftp, localPath, remotePath) {
    this.sftp = sftp;
    this.targetFiles = []; //待上传的所有文件
    this.localPath = localPath;
    this.remotePath = remotePath;
  }

  //将单文件路径同一处理为｛filename:xx,filePath:xxx}格式的对象，并且存入targetFiles中
  async normalizeSinglePath(filePath) {
    const filename = path.basename(filePath);
    const fileDirectoryPath = path.dirname(filePath);
    const fileObject = {
      filename,
      filePath
    };
    const pathItem = this.targetFiles.find(item => item.path === fileDirectoryPath);
    if (pathItem) {
      pathItem.files.push(fileObject);
    } else {
      this.targetFiles.push({
        path: fileDirectoryPath,
        files: [fileObject]
      });
    }
  }

  //处理本地文件的文件路径
  async initFilePaths(targetPath = localPath) {
    const stats = await promiseFs.stat(targetPath);
    if (stats.isDirectory()) {
      const files = await promiseFs.readdir(targetPath);
      const promises = [];
      files.forEach(file =>
        promises.push(this.initFilePaths(path.resolve(targetPath, file)))
      );
      await Promise.all(promises);
    } else {
      this.normalizeSinglePath(targetPath);
    }
  }

/**
 * 将目标文件路径转为相对跟路径的相对路径数组
 * 例如rootPath: /document, targetFilePath: /document/work/wednesday/morning,
 * 返回： [work,wednesday,morning]
 * @param {*} rootPath 基准路径
 * @param {*} targetFilePath 目标路径
 */
  relativeToPath(rootPath, targetFilePath) {
    if (!targetFilePath.includes(rootPath)) {
      return console.log('文件路径不在指定目录内');
    }
    const newPath = targetFilePath.replace(rootPath, '/');
    const pathsArr = newPath.split(path.sep);
    return pathsArr;
  }

  async uploadFile(filePath, remotePath) {
    return new Promise((resolve, reject) => {
      this.sftp.fastPut(filePath, remotePath, err => {
        if (err) {
          console.log(err);
          reject(err);
        } else {
          resolve();
        }
      });
    });
  }

  // 创建远程目录
  async mkdir(dirPath) {
    const dirs = this.relativeToPath(localPath, dirPath);
    const _mkdir = convertPromise(this.sftp.mkdir, this.sftp);
    let rPath = this.remotePath;
    //根据相对路径数组，一级一级的创建目录，如a/b/c目录，则必须先有a才能创建b，不然会报错
    for (let dir of dirs) {
      if (dir !== '/') {
        rPath += `/${dir}`;
      }

      //检查需要创建的目录是否存在，如果不存在则需要创建
      try {
        await checkPathInFtp(this.sftp, rPath);
      } catch (e) {
        try {
          await _mkdir(rPath);
        } catch (e) {
          console.log(`创建目录${rPath}失败：${e}`);
          return false;
        }
      }
    }
    return rPath;
  }

  //上传所有本地文件
  async startUpload() {
    try {
      await checkPathInFtp(this.sftp, this.remotePath);
      //先对远程服务器上的文件目录进行备份
      await backup(this.sftp, this.remotePath);
    } catch (e) {
      console.log('备份失败：备份文件不存在')
    }

    try {
      await this.initFilePaths();
      for (let item of this.targetFiles) {
        let remotePath = await this.mkdir(item.path);
        if (remotePath) {
          const promises = [];
          item.files.forEach(file => {
            promises.push(this.uploadFile(file.filePath, `${remotePath}/${file.filename}`));
          });
          await Promise.all(promises);
        } else {
          throw new Error('创建服务器目录失败');
        }
      }
      //成功上传后删除备份文件
      await deleteFileInFtp(this.sftp, `${this.remotePath}${FIX_FILENAME}`);
      console.log('上传服务器成功');
    } catch (e) {
      console.error(`上传服务器失败：${e}`);
      await rollback(this.sftp, this.remotePath); //上传失败后回滚
      return;
    }
  }
}

var conn = new Client();
conn.on('ready', function() {
  console.log('Client ready');
  conn.sftp(function(err, sftp) {
    if (err) throw err;
    new UpFile(sftp, localPath, remotePath).startUpload().then(() => {
      conn.end();
    });
  });
});

conn.connect({
  host: serverConfig.host,
  port: serverConfig.port,
  username: serverConfig.username,
  password: serverConfig.password,
  tryKeyboard: true
});