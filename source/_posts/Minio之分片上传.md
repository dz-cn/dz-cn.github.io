---
title: Minio之分片上传
tags:
  - default
description: |
  URL预签名实现分片上传与断点续传
pinned: 0
lang: zh
categories:
  - default
type: post
wordCount: 570
charCount: 7649
imgCount: 0
vidCount: 0
wsCount: 5
cbCount: 4
readTime: 约5分钟
date: 2026-04-10 11:35:06
---
<!-- toc -->
# Minio 之分片上传

## 前言
关于分片上传，每个开发人员都有所耳闻。但是，问到具体的细节，相信也会不知所措。本期，就给你揉碎了讲讲，利用 Minio 的URL预签名策略，实现分片上传的机制。

## 自动SDK <span alt="shake">VS</span> 手动URL

### 1、自动SDK
```java
// 自动SDK
MINIO_CLIENT.uploadObject()
```
相信很多人都没有在意，一行短短的代码，内部却实现了当文件过大时，自动分片上传的逻辑，以及断点续传的逻辑。没错，新版<9.0.0>的 Minio 是没有直接提供分片上传的方法的。但是也将普通的文件上传集成到了一个里面，简化了开发，提高了效率，让开发者无需为了分片操心，对开发者可谓是无微不至。

但是这样仍然带来了问题。由于隐藏了分片逻辑，导致开发者无法感知分片的进度，同时，分片和上传都是在后端进行的，因此极大的消耗了服务器的带宽和CPU的资源。当并发量较大的时候，有可能导致CPU连续高强度的工作而出现服务卡顿的现象。

### 2、手动URL
“手动RUL”，本质上就是利用 URL 的预签名上传链接，自己实现一套分片上传的方法。通过设置与预签名的有效期、文件大小，来确保上传链接的安全性。前端利用URL上传分片，保证后端只需提供一次URL，保证后端服务的高可用性，同时有利于前端监控上传进度，给用户更好的体验。最后前端通过重试回调逻辑，保证过期URL的重新请求。
亲测解决以下五大问题：
- 文件不存在、分片不存在
- 文件存在、分片全部不存在
- 文件存在、分片部分不存在
- 文件不存在、分片全部存在
- 文件不存在、分片部分存在

## 前端部分关键源码
```js
try {
        // 计算整个文件的md5
        const fileMd5 = await calculateMD5(file);
        // 计算每个分片的信息
        const chunks = await shardFile(file);

        const suffix = file.name.substring(file.name.lastIndexOf('.') + 1);
        const progressBar = fileDiv.querySelector('.progress');
        const progressText = fileDiv.querySelector('.progress-text');

        let len = chunks.length;

        fileDiv.style.display = 'block';
        preparingDiv.remove();

        // 上来先尝试合并一次(解决分片已存在的问题)
        let list = await fetch('', {
            method: 'PUT'
        }).then(response => response.json().then(date => date))

        if(list != null && list.length === 0) {
            console.log('所有分片全部存在,已将其合并!')
            updateProgress(progressBar, progressText, 100);
            return
        }

        // 构建请求参数
        const chunkFormData = new FormData();
        chunkFormData.append('md5File', fileMd5);
        chunkFormData.append('chunkIds', list);
        chunkFormData.append('suffix', suffix);

        // 记录开始时间
        const time = Date.now();

        // 上传并合并
        let result = await handleFile(list,chunkFormData,len,chunks,progressBar,progressText,fileMd5,suffix)

        // 回调判断,并合并
        let count = 5;
        while(result != null && result.length > 0 && count > 0) {
            console.log('文件合并失败，正在检查分片并重新上传...', count)
            result = await handleFile(list,chunkFormData,len,chunks,progressBar,progressText,fileMd5,suffix)
            count --;
        }

        if(result != null && result.length > 0 && count > 0) {
            console.log('文件合并失败，请重新上传')
            return
        }

        console.log(`文件合并成功`);
        updateProgress(progressBar, progressText, 100);

    } catch (error) {
        console.error('文件处理失败:');
    }
```

```js
async function handleFile(list, chunkFormData, len, chunks, progressBar, progressText, fileMd5, suffix) {
    // 获取上传 URL
    const urls = await fetch(``, {
        method: 'PUT',
        body: chunkFormData
    }).then(response => response.json().then(date => date.urls));

    if(urls == null) {
        // 文件已经存在---秒传
        return []
    }

    // 逐个分片上传
    for(let i = 0;i < list.length;i++) {
        const index = list[i];
        const url = urls[i]
        const time = Date.now();

        await fetch(url, {
            method: 'PUT',
            body: chunks[index-1].blob
        });


        console.log(`分片上传成功`);

        const progress = (i / len * 99).toFixed(2);
        updateProgress(progressBar, progressText, progress);
    }

    // 合并文件
    const result = await fetch(``, {
        method: 'PUT'
    }).then(response => response.json().then(date => date));

    console.log("合并返回结果: ",result)
    return result
}
```



## 后端源码
```java
package dz.cn.minio.util;

import io.minio.*;
import io.minio.errors.ErrorResponseException;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.util.DigestUtils;
import org.springframework.util.FileSystemUtils;
import javax.annotation.Resource;
import java.io.BufferedInputStream;
import java.io.FileDescriptor;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;

/**
 * ClassName:MinioFileUtil
 * Description:
 * @Author:独酌
 * @Create:2024/12/8 11:21
 */
@Slf4j
@Component
public class MinioFileUtil {

    /*
      Minio的连接对象
    */
    @Getter //增加可扩展、自定义性，用于获取连接对象，直接用类名掉用 【MinioFileUtil.getMINIO_CLIENT()】
    private static final MinioClient MINIO_CLIENT;
    private static final String BUCKET = "dz-test";
    private static final String BUCKET_TEMP = "temp";
    private static final String HOST = "http://192.168.59.128:9000";
    private static final String ACCESS_KEY = "minioadmin";
    private static final String SECRET_KEY = "minioadmin";

    static {
        MINIO_CLIENT = MinioClient.builder()
                .endpoint(HOST)
                .credentials(ACCESS_KEY, SECRET_KEY)
                .build();
        try {
            initBucket();
            log.info("✅ Minio 初始化成功");
        } catch (Exception e) {
            log.error("Minio连接失败！",e);
        }
    }


    /**
     * 普通的使用 minio 上传文件
     *
     * @param file     文件流
     * @param fileSize 文件名 大小
     * @param path     文件储存的路径（xxx/xxx/）
     * @param suffix   文件后缀
     * @return 文件的存储路径，支持浏览器的图片预览
     */
    public String ordinaryUploadFile(InputStream file, long fileSize, String path, String suffix, String contentType) {
        try {
            // 获取文件的hash值
            BufferedInputStream bis = new BufferedInputStream(file);
            bis.mark(Integer.MAX_VALUE);
            String md5 = DigestUtils.md5DigestAsHex(bis);
            String name = md5 + suffix;
            bis.reset(); // 重置文件流，以便后续的上传
            // 判断存不存在
            if(isExistObject(path + name)){
                log.info("此文件已经上传过了");
                return null;
            }
            MINIO_CLIENT.putObject(
                    PutObjectArgs.builder()
                            .bucket(BUCKET)
                            .contentType(contentType)
                            .stream(bis, fileSize, -1L)
                            .object(path + name)
                            .build());
            log.info("单文件上传成功");
            return HOST + "/" + BUCKET +  "/" + path + name;
        } catch (Exception e) {
            log.error("单文件上传失败",e);
        }
        return null;
    }


    /**
     * 初始化分片上传,并获取URL
     * @param fileMD5 文件的MD5
     * @param suffix 后缀名
     * @return 一次性将所有需要上传的URL返回给前端
     */
    public List<String> initMultipart(String fileMD5,List<Integer> chunkIds, String suffix) {
        boolean existObject = isExistObject(fileMD5 + "." + suffix);
        if(existObject){
            log.info("秒传");
            return null;
        } else {
            try {
                List<String> urls = new ArrayList<>();
                for(int i : chunkIds) {
                    String objectName = fileMD5 + ":" + i + "." + suffix;
                    // 返回所需的全部 URL
                    String objectUrl = MINIO_CLIENT.getPresignedObjectUrl(
                            GetPresignedObjectUrlArgs.builder()
                                    .bucket(BUCKET_TEMP)
                                    .method(Http.Method.PUT)
                                    .object(objectName)
                                    .expiry(1, TimeUnit.HOURS)
                                    .build()
                    );
                    urls.add(objectUrl);
                }
                return urls;
            } catch (Exception e) {
                log.error("获取全部的URL失败",e);
            }
            return null;
        }
    }

    /**
     * 合并分片
     * @param fileMD5 整个文件MD5
     * @param suffix 文件后缀
     * @param shardCount 分片总数
     * @return 返回不存在的分片 id
     */
    public List<Integer> mergeMultipart(String fileMD5, String suffix, Integer shardCount) {
        try {
            String objectName = fileMD5 + ":" + shardCount + "." + suffix;
            // 先判断文件是否存在
            boolean existObject = isExistObject(objectName);
            if(existObject) {
                log.info("此文件已经上传过了");
                return new ArrayList<>();
            }

            // 检查是否所有分片全部存在
            List<Integer> list = new ArrayList<>();
            for(int i = 1;i <= shardCount;i++) {
                try {
                    MINIO_CLIENT.statObject(StatObjectArgs.builder()
                            .bucket(BUCKET_TEMP)
                            .object(fileMD5 + ":" + i + "." + suffix)
                            .build()
                    );
                } catch (ErrorResponseException e) {
                    log.warn("分片<{}>不存在",i);
                    list.add(i);
                }
            }

            if(!list.isEmpty()) return list;

            List<SourceObject> sources = new ArrayList<>();
            for(int i = 1;i <= shardCount;i++) {
                SourceObject object = SourceObject.builder()
                        .bucket(BUCKET_TEMP)
                        .object(fileMD5 + ":" + i + "." + suffix)
                        .build();
                sources.add(object);
            }
            MINIO_CLIENT.composeObject(
                    ComposeObjectArgs.builder()
                            .bucket(BUCKET)
                            .object(objectName)
                            .sources(sources)
                            .build()
            );
            log.info("合并分片成功，访问地址为: {}",HOST + "/" + BUCKET +  "/" + objectName);
            return list;
        } catch (Exception e) {
            log.error("合并分片失败",e);
        }
        return null;
    }

    /**
     * 创建桶
     */
    private static void initBucket() {
        try{
            boolean isExist = MINIO_CLIENT.bucketExists(BucketExistsArgs.builder().bucket(BUCKET).build());
            isExist = isExist && MINIO_CLIENT.bucketExists(BucketExistsArgs.builder().bucket(BUCKET_TEMP).build());
            if(!isExist) {
                MINIO_CLIENT.makeBucket(MakeBucketArgs.builder().bucket(BUCKET).build());
                MINIO_CLIENT.makeBucket(MakeBucketArgs.builder().bucket(BUCKET_TEMP).build());
            }
        } catch (Exception e) {
            log.warn("{}：桶创建失败：{}", Thread.currentThread().getName(), e.getMessage());
        }
    }

    /**
     * 判断指定桶中的文件是否存在
     *
     * @param objectName 对象名
     * @return true 存在，false 不存在
     */
    private boolean isExistObject(String objectName){
        try{
            MINIO_CLIENT.statObject(StatObjectArgs.builder()
                    .bucket(MinioFileUtil.BUCKET)
                    .object(objectName)
                    .build());
        } catch (Exception e){
            return false;
        }
        return true;
    }

    /**
     * 获取下载链接
     * @param path 文件的存储路径 xxx/xxx/
     * @param name 文件名
     * @param suffix 文件后缀
     * @return 下载链接
     */
    public String getObjectURL(String path,String name,String suffix){
        try {
            return MINIO_CLIENT.getPresignedObjectUrl(GetPresignedObjectUrlArgs.builder()
                    .object(path + name + suffix)
                    .bucket(BUCKET)
                    .method(Http.Method.GET)
                    .expiry(15, TimeUnit.MINUTES) //有效期为 15 分钟
                    .build());
        } catch (Exception e){
            log.error(Thread.currentThread().getName() + "：获取下载链接失败!",e);
        }
        return null;
    }
}
```
