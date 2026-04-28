## Unity Addressables 资源加载器开发全手册 (全平台通用版)
## 一、 Addressables 系统简介
Addressables (可寻址资源系统) 是 Unity 官方推荐的高级资源管理框架，旨在取代传统的 Resources 和手动 AssetBundle 管理。

* 解耦加载与路径：通过“地址（Key）”加载，无需关心资源存储位置。
* 内存引用计数：内置内存管理，通过引用计数自动追踪资源的加载与释放。
* 一键热更新：支持将资源部署在 CDN，实现内容的在线增量更新。

------------------------------
## 二、 环境准备与资源配置

   1. 安装插件：Window -> Package Manager -> 搜索并安装 Addressables。
   2. 标记资源：在 Inspector 顶部勾选 Addressable，并修改 Key（如 Hero_Knight）。
   3. 资源分组 (Groups)：
   * Local：随 App 安装包发布。
      * Remote：放在 CDN 上，运行游戏时按需下载。
   
------------------------------
## 三、 设置 CDN 远程地址与热更新## 1. 配置 Profile
在 Addressables Profiles 窗口：

* Remote.LoadPath: 修改为你的 CDN URL（例：https://yourgame.com[BuildTarget]）。

## 2. 开启远程目录 (Catalog)
在 AddressableAssetSettings 中勾选 Build Remote Catalog。这让客户端能通过索引文件发现服务器上的新资源。
------------------------------
## 四、 Hash 缓存与存储机制

* 持久化存储：下载过的资源保存在本地硬盘，重启游戏不会丢失。
* Hash 指纹校验：每个包都有一个 .hash 文件（版本指纹）。
* 匹配：本地 Hash == 服务器 Hash $\rightarrow$ 直接读取硬盘（秒开，省流量）。
   * 不匹配：本地 Hash $\neq$ 服务器 Hash $\rightarrow$ 自动下载新包并替换旧缓存。

------------------------------
## 五、 微信小程序/小游戏特殊规则
在微信小游戏（WebGL）环境使用 Addressables 时，需遵循微信平台的特殊逻辑：
## 1. 缓存限制 (200MB)

* 自动托管：微信会自动拦截下载请求并缓存资源，但本地缓存上限通常为 200MB。
* 自动清理 (LRU)：当资源超过 200MB 时，微信会根据“最近最少使用”原则自动删除旧资源。这意味着旧章节资源可能被系统自动清理，回玩时需重新下载。

## 2. 网络要求

* 域名白名单：CDN 地址必须是 HTTPS，且必须在微信公众平台配置为“下载域名白名单”。
* 并发限制：微信环境下载并发数有限，建议 Bundle 大小控制在 1MB - 2MB 以提高加载成功率。

------------------------------
## 六、 存储空间管理 (磁盘瘦身)## 1. 清理废弃资源

// 清理所有不再被当前 Catalog 记录的旧版本缓存
Addressables.ClearDependencyCacheAsync(null); 

## 2. 清理旧章节资源 (基于 Label)

// 释放特定标签（如 Chapter1）占用的磁盘空间
Addressables.ClearDependencyCacheAsync("Chapter1");

------------------------------
## 七、 资源下载器封装 (ResManager)

using System;using System.Collections.Generic;using UnityEngine;using UnityEngine.AddressableAssets;using UnityEngine.ResourceManagement.AsyncOperations;
public class ResManager : MonoBehaviour
{
    public static ResManager Instance { get; private set; }
    private Dictionary<string, AsyncOperationHandle> _assetHandles = new Dictionary<string, AsyncOperationHandle>();

    private void Awake()
    {
        if (Instance == null) { Instance = this; DontDestroyOnLoad(gameObject); }
        else { Destroy(gameObject); }
    }

    // 加载资源 (RAM)
    public void LoadAsset<T>(string key, Action<T> onComplete) where T : UnityEngine.Object
    {
        if (_assetHandles.TryGetValue(key, out AsyncOperationHandle handle))
        {
            if (handle.IsDone && handle.Status == AsyncOperationStatus.Succeeded)
            {
                onComplete?.Invoke(handle.Result as T);
                return;
            }
        }
        Addressables.LoadAssetAsync<T>(key).Completed += (op) =>
        {
            if (op.Status == AsyncOperationStatus.Succeeded)
            {
                if (!_assetHandles.ContainsKey(key)) _assetHandles.Add(key, op);
                onComplete?.Invoke(op.Result);
            }
            else { Debug.LogError($"加载失败: {key}"); Addressables.Release(op); }
        };
    }

    // 卸载资源 (RAM)
    public void UnloadAsset(string key)
    {
        if (_assetHandles.TryGetValue(key, out AsyncOperationHandle handle))
        {
            Addressables.Release(handle);
            _assetHandles.Remove(key);
        }
    }
}

------------------------------
## 八、 总结建议

   1. 微信平台：利用 [微信小游戏 Unity 适配面板] 插件来辅助管理 200MB 缓存，效果更佳。
   2. 成对原则：每次 LoadAsset 必须对应一次 UnloadAsset，确保运行内存 (RAM) 不泄漏。
   3. 硬盘清理：对大体量游戏，建议按章节（Label）在适当的时机手动执行磁盘清理。


