---
title: 深入探究 Android Content Provider 安全
categories: [Android]
layout: post
---

## Content Provider 通信模型

Android Content Provider 采用 C/S 架构，通信过程可粗略概括为

App. A use ContentResolver  -- to communicates -->   ContentProvider implemented by App. B

[![content_provider_class_model](/assets/img/post/2021/content-provider-class-model.jpg)](https://blog.csdn.net/u010753159/article/details/51900318)

其中，客户端和服务端的通信基于 Binder：

> 在建立 Binder 通信之前，客户端通过 AMS 获取到服务端的 Binder，具体过程可概括为：客户端从 URI 从提取 CP 的 authorities （CP 的唯一标识），向 AMS 调用 getContentProvider 方法，AMS 收到客户端的 Content Provider 请求后根据 CP 的名称（authorities）向 PMS 查询该 CP 的信息（查询到后会缓存），AMS 根据信息启动 CP 服务端进程，并远程调用 scheduleInstallProvider 通知服务端准备和发布目标 CP，服务端完成 installProvider 后最终反向调用 AMS 的 publishContentProviders 把 CP 注册进来，最后 AMS 再根据条件（权限校验等）把注册进来的 CP 的 Binder 发送给客户端，这样客户端就能和 CP 服务端通信了。[参考](http://gityuan.com/2016/07/30/content-provider/)

![content_provider_ipc](/assets/img/post/2021/content_provider_ipc.jpeg)

其中 [ContentProvider](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/content/ContentProvider.java) 是服务端要继承的抽象类，其内部定义了 query、delete 等等抽象方法，须要去实现、重写。此外其通过内部成员变量 mTransport 持有内部类 Transport 对象（负责 Binder IPC 通信）：

```java
ContentProvider.java

private Transport mTransport = new Transport();

...略

    /**
     * Binder object that deals with remoting.
     *
     * @hide
     */
    class Transport extends ContentProviderNative {

```

Transport 继承自 [ContentProviderNative](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/android/content/ContentProviderNative.java)，而打开 ContentProviderNative 可以看到其继承 Binder 类、实现 IContentProvider 接口，可见是很典型的 Binder 通信模型代码：Transport 是实现 IContentProvider 接口的 Binder 对象，负责处理来自客户端的 Binder 请求以及作为Binder句柄发送给客户端使用。

打开  [Transport.query()](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/content/ContentProvider.java#227)，可以看到先执行了权限检查等逻辑，然后调用 mInterface.query() ，即 ContentProvider 的 query() 、即应用程序的 query() 实现。

## Content Provider 注册与获取

### 注册 CP

PMS 中 的 PackageParser 的 parseProvider 解析 AndroidManifest 中的 CP，生成 ProviderInfo 供后续获使用

### 获取 CP

AMS 先初步检查权限（不含 AppOps），如果通过返回 Binder，CP 服务端所在进程的 ContentProvider.java 再二次根据接口类型做最终权限检查和 AppOps 检查。

详细见后文。

## 静态 Content  Provider 权限

可以对 Content Provider 整个（读写一起）配置权限，也可以细粒度地，对Content Provider读写分别配置权限（readPermission、writePermission），甚至针对 path 配置权限，当 Path 权限和 Global 权限同时存在时，前者[优先于后者](https://developer.android.com/guide/topics/providers/content-provider-creating.html#implementing-permissions)。 [参考](https://developer.android.com/guide/topics/providers/content-provider-creating.html#Permissions)。

此外，Android Framework 不同的 Content Provider 接口执行权限检查情况不同，有的会校验写权限（如 insert），有的会校验读权限，而有的则不校验任何权限，而 getType 是唯一的甚至允许 exported=false 时可被[外部调用](https://developer.android.com/reference/android/content/ContentProvider#getType(android.net.Uri)) 的接口，列表如下：

||insert|delete|update|query|call|getStreamTypes|bulkInsert|getType|openFile  r|opnFile  w|
|:-:|:-:|:-:|:-:|---|---|:-:|:-:|---|:-:|:-:|
|**readPermission**|no|no|no|yes|no|no|no|no|yes|no|
| **writePermission** |yes|yes|yes|no|no|no|yes|no|no|yes|

注意：除了 getType 外，上表中 read/write 权限都不需要的，并不意味着客户端就可以任意访问到，参见[Part 1. AMS 对 Content Provider 调用的权限检查](#part-1-ams-对-content-provider-调用的权限检查)。

Content Provider 服务要实现只读，可以配置 writePermission 或相应接口实现时 [return 0](https://developer.android.com/guide/topics/providers/content-provider-creating.html#ContentProvider)

## 动态 Content Provider 权限

根据 [Android 官方描述] (https://developer.android.com/training/articles/security-tips#ContentProviders)，如果应用想动态地授予外部应用 Content Provider 权限（即使该Content Provider 的 exported 属性为 false 或要求了应用不满足的权限），必须在 Provider 的 Manifest 中声明 android:grantUriPermissions 为 true 的属性，使所有 URI 都能够被动态授权，默认为 false；或者仅开放有限的 path 可被动态授权，做法是在其下增加 <grant-uri-permission>

```xml
<grant-uri-permission android:path="string"
                      android:pathPattern="string"
                      android:pathPrefix="string" />
```

注 1：Android 的 grantUriPermissions 特性要求基于一个 exported=false 的 Content Provider，exported=true 的 Content Provider 外界总是能访问。HMSCore 的 StubContentProvider exported = true，调用 Context().checkUriPermission 试图校验 CP 时，无论先前是否已 grant URI 权限，该接口总是返回**有权限**。

注 2：Android 应用框架会用 [ProviderInfo#uriPermissionPatterns](https://developer.android.com/reference/android/content/pm/ProviderInfo#uriPermissionPatterns)  记录 Provider 允许动态授权的 URI 列表，不在此列表的 URI 无法被动态授权；应用可以通过 [Context.checkUriPermission() ](https://developer.android.com/reference/android/content/Context#checkUriPermission(android.net.Uri,%20java.lang.String,%20java.lang.String,%20int,%20int,%20int)) 查询调用者是否被动态授予了目标 URI 的权限。

这样一来，Content Provider 就可以通过：

1. 调用 [Context.grantPermission()](https://developer.android.com/reference/android/content/Context#grantUriPermission(java.lang.String,%20android.net.Uri,%20int)) 授予外部应用（指定包名）某个 URI 的访问权限。对应的 revokeUriPermission() 来撤销某个 Uri 的全部访问权限或指定应用的访问权限

2. （实践中更常用）在发送给外部应用的 Intent 中设置 FLAG_GRANT_READ_URI_PERMISSION/FLAG_GRANT_WRITE_URI_PERMISSION 来授权某个 URI 的访问权限，比如邮件应用临时授予（通过 startActivityForResult）图片浏览应用访问自己的附件图片的权限。

二者差别：

前者的授权可以通过参数（toPackage、uri、modeFlags）控制授权者包名、Uri、读写权限组合、授权[是否持久化](https://developer.android.com/reference/android/content/Intent#FLAG_GRANT_PERSISTABLE_URI_PERMISSION)等（默认情况下手机重启或手动 revoke 就会使授权失效）； 
后者则仅在接收授权的应用的任务栈（task）存续期间存在，一旦 task 销毁，授权也会自动失效，此时 Content Provider 不需要再手动 revoke。

上述权限机制的实现在 ActivityManangerService 里面，其中持久化授权记录在 /data/system/urigrants.xml 文件中; 持久化授权还需要被授权者真实存在，因此要求接收授权者调用 ContentResolver#takePersistableUriPermission(Uri, int) 使持久化授权真实生效。

详细参考：

- [Content provider basics](https://developer.android.com/guide/topics/providers/content-provider-basics#getting-access-with-temporary-permissions)
- [Android Security Internals: An In-Depth Guide to Android's Security Architecture](https://books.google.co.jp/books?id=-QcvDwAAQBAJ&pg=PA48&lpg=PA48&dq=grantUriPermissions+security&source=bl&ots=L9hsYT45tA&sig=ACfU3U0U5uPcwwwgaR-GbGFOj_qmtABQmA&hl=zh-CN&sa=X&ved=2ahUKEwiDnY_Wiv7oAhUxNKYKHc37DMEQ6AEwAnoECAgQAQ#v=onepage&q=grantUriPermissions%20security&f=false)

## Contet Provider 权限检查的实现

- 客户端： ContentResolver、ContextImpl.java、 ActivityThread.java
- AMS：ActivityManagerService.java、ActivityManager.java
- 服务端：ContentProvider.java

Content Provider 的权限校验实现分为两部分，第一部分是 AMS 实现的

### Part 1. AMS 对 Content Provider 调用的权限检查

服务端注册调用 AMS.publishContentProviders() 接口注册 CP，使得客户端应用能够通过 CP 名称查询获取到 CP。该接口除了使用 enforceNotIsolatedCaller() 限制沙箱应用调用外（很多接口都有这一限制），没有额外的权限限制，。

```java
    /* package */ void enforceNotIsolatedCaller(String caller) {
        if (UserHandle.isIsolated(Binder.getCallingUid())) {
            throw new SecurityException("Isolated process not allowed to call " + caller);
        }
    }
```

客户端查询 Conent Provider 时（调用 acquireProvider、...、AMS.getContentProvider） AMS 会调用 PMS 的接口，综合服务端 CP 声明的权限、客户端的权限、服务端 CP exported 情况决定是否返回服务端的 Binder 句柄。具体实现在 AMS.getContentProviderImpl() 中调用 [checkContentProviderPermissionLocked](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-11.0.0_r25/services/core/java/com/android/server/am/ActivityManagerService.java#6837)

```java
    /**
     * Check if {@link ProcessRecord} has a possible chance at accessing the
     * given {@link ProviderInfo}. Final permission checking is always done
     * in {@link ContentProvider}.
     */
    private final String checkContentProviderPermissionLocked(
            ProviderInfo cpi, ProcessRecord r, int userId, boolean checkUser) { // cpi 是服务端 provider 的信息
        final int callingPid = (r != null) ? r.pid : Binder.getCallingPid();
        final int callingUid = (r != null) ? r.uid : Binder.getCallingUid();
        boolean checkedGrants = false;
        if (checkUser) {
            // Looking for cross-user grants before enforcing the typical cross-users permissions
            int tmpTargetUserId = mUserController.unsafeConvertIncomingUser(userId);
            if (tmpTargetUserId != UserHandle.getUserId(callingUid)) {
                if (mUgmInternal.checkAuthorityGrants(
                        callingUid, cpi, tmpTargetUserId, checkUser)) {
                    return null;
                }
                checkedGrants = true;
            }
            userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, false,
                    ALLOW_NON_FULL, "checkContentProviderPermissionLocked " + cpi.authority, null);
            if (userId != tmpTargetUserId) {
                // When we actually went to determine the final targer user ID, this ended
                // up different than our initial check for the authority.  This is because
                // they had asked for USER_CURRENT_OR_SELF and we ended up switching to
                // SELF.  So we need to re-check the grants again.
                checkedGrants = false;
            }
        }
        // 第一次机会，检查组件级别权限
        // 如果组件的 r/w 任意一个权限显式满足或任意一个为 null，就代表客户端有机会访问服务端，返回成功。否则，看下 path 权限有没有机会。
        if (checkComponentPermission(cpi.readPermission, callingPid, callingUid,
                cpi.applicationInfo.uid, cpi.exported)
                == PackageManager.PERMISSION_GRANTED) {
            return null;
        }
        if (checkComponentPermission(cpi.writePermission, callingPid, callingUid,
                cpi.applicationInfo.uid, cpi.exported)
                == PackageManager.PERMISSION_GRANTED) {
            return null;
        }
        // 第二次机会，path 权限
        PathPermission[] pps = cpi.pathPermissions;
        if (pps != null) {
            int i = pps.length;
            // 将所有 path 的 readPermission  writePermission 权限都检查一遍，只要有任意一个权限显式满足（且 path 权限非 null），则返回成功。否则下一步。这里 path 权限为 null 时，并不视为「机会」，当组件权限显式拒绝时，这才是合理的。
            while (i > 0) {
                i--;
                PathPermission pp = pps[i];
                String pprperm = pp.getReadPermission();
                if (pprperm != null && checkComponentPermission(pprperm, callingPid, callingUid,
                        cpi.applicationInfo.uid, cpi.exported)
                        == PackageManager.PERMISSION_GRANTED) {
                    return null;
                }
                String ppwperm = pp.getWritePermission();
                if (ppwperm != null && checkComponentPermission(ppwperm, callingPid, callingUid,
                        cpi.applicationInfo.uid, cpi.exported)
                        == PackageManager.PERMISSION_GRANTED) {
                    return null;
                }
            }
        }
        // 最后一个机会：检查 grant uri，看该客户端是否被服务端 grant uri 过
        if (!checkedGrants
                && mUgmInternal.checkAuthorityGrants(callingUid, cpi, userId, checkUser)) {
            return null;
        }
        final String suffix;
        if (!cpi.exported) {
            suffix = " that is not exported from UID " + cpi.applicationInfo.uid;
        } else if (android.Manifest.permission.MANAGE_DOCUMENTS.equals(cpi.readPermission)) {
            suffix = " requires that you obtain access using ACTION_OPEN_DOCUMENT or related APIs";
        } else {
            suffix = " requires " + cpi.readPermission + " or " + cpi.writePermission;
        }
        final String msg = "Permission Denial: opening provider " + cpi.name
                + " from " + (r != null ? r : "(null)") + " (pid=" + callingPid
                + ", uid=" + callingUid + ")" + suffix;
        Slog.w(TAG, msg);
        return msg;
    }
```

其中入参 permission 是目标组件声明的权限，owningUid 是目标组件应用的 UID， exported 是目标组件的暴露情况，uid 是调用者的 UID。这里和 AMS/PMS 所有权限检查代码一样，只检查传统权限模型中的 Permission，而不检查 **App Ops**。checkContentProviderPermissionLocked 的逻辑总的来说是：只要调用者有任何访问 CP 的机会就返回成功，允许将服务端的 Binder 发送给客户端（正如方法开头的注释所说，当然服务端还有下半场权限检查），具体来说：

0. 默认无访问权限，依次检查下列权限规则，当任意「机会」存在时返回权限检查成功（意味着 AMS 将返回 CP 服务端的 Binder 给到客户端应用）

1. 检查跨（多）用户的权限，这里省略。

2. 服务端组件并不要求权限或客户端显式具有服务端 ContentProvider 组件维度声明的权限，视为有「机会」。否则下一步。

    **如果组件的 r/w 任意一个权限显式满足或任意一个为 null，就代表客户端有机会访问服务端，返回成功。—— 类比于一套房子的若干大门，和内部的房间门，当任意大门是开的（权限 null 或者显式满足），一定是有机会进入房子**

    分别调用 checkComponentPermission()，先后检查客户端是否具有服务端件声明的 `readPermission` 和 `writePermission` 权限，检查客户端是否 PERMISSION_GRANTED。

    注意：这里 AMS 对组件级的权限检查并不直接检查 ContentProvider 清单文件中声明的 `permission` 属性 (整体权限)。实际上，AMS 内记录实例权限信息的 ProviderInfo 类也不存在对应于清单中 `permission` 属性的字段，这是因为 [PackageParser.parseProvider()](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-11.0.0_r25/core/java/android/content/pm/PackageParser.java#5065) 解析 ContentProvider 组件时，权限信息会拍扁为 ProviderInfo.readPermission 和 ProviderInfo.readPermission，其取值遵循清单文件中 ContentProvider r/w  权限优先于组件权限的原则：当清单中 r、w 非空时，取 r、w 的值；当 r 或 w 为空无法取到值时，对应地以 permission 的值为准（包括空，即最终取值 null）。参考文档 [provider-element](https://developer.android.com/guide/topics/manifest/provider-element#prmsn) 

    checkComponentPermission 的实现在 [ActivityManager.checkComponentPermission()](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-11.0.0_r25/core/java/android/app/ActivityManager.java#4213) ，会检查 exported 和权限拥有情况（其中权权限检查是调用 PMS 的接口）。当 exported = false 时直接返回失败，exported = true 时才检查权限，并且如下代码可以验证：当服务端声明的权限为 null 时返回 PERMISSION_GRANTED，即无权限保护时即可访问 ContentProvider（准确说还只是获取 Binder 句柄）。

    ```java
    if (permission == null) { //服务端权限，这边是 readPermission 或 writePermission
        return PackageManager.PERMISSION_GRANTED;
    }
    ```

3. 客户端显式具有服务端 pathPermissions 声明的权限的集合中任意一个，则视为有「机会」。—— 虽然大门是紧闭的，但只要有开窗，还是有机会进入房子，当然空 path 权限等价于没有窗户。

    组件粒度权限检查失败，开始检查 pathPermission。pathPermission 是针对单个 path 设置的权限，但 AMS 的检查并不看具体的 URI ，只要客户端具有任意一个权限即可（针对 URI 的检查，放在 ContentProvider.java）。还是使用 checkComponentPermission ，因此检查结果仍然包含 exported 属性。如果检查成功则返回成功，否则下一步。

    注意：到这一步，空 pathpermission 不能像空组件权限一样视为机会了。

4. 检查 grant uri，看该客户端是否被服务端 grant uri 过，如果满足，则给最后一次「机会」。

    pathPermissions 也检查失败，还有机会，检查客户端是否被服务端 grant uri 过，只要 grant 过即返回成功，不管是什么 URI。

    ```java
    if (!checkedGrants
             && mUgmInternal.checkAuthorityGrants(callingUid, cpi, userId, checkUser)) {
        return null;
    }
    ```

逻辑是：检查 callingUid 所有的 granted 权限的 URI 中，是否有任意一个 URI  其 content provider Authority（通过 uri.getAuthority() 获取） 能匹配目标 content provider，如果匹配说明这个 callingUid 被目标 cp grant 过。如果匹配则返回成功。否则下一步（到这里就是失败了）。这里也是为什么 exported 为 false 的 cp 认然有机会被访问的原因。

上述权限检查逻辑似乎哪里不对：Content Provider 不同接口权限要求是不一样的，比如 query 和 insert 一个是要求符合 readPermission，而 insert 则是 writePermission，而且也没有针对具体的 URI 做区分。

原来这里 AMS 只是决定 Binder 句柄是否返回给客户端，只是表示客户端可能具有权限，客户端获取到 Binder 句柄建立通信后，在服务端进程运行的 Android 应用框架 ContentProvider 类还会执行决定性的权限检查。对于任何一个权限都不具有不沾边的，就直接拒绝了。在一开始的注释也有说明。

### part2. ContentProvider.java 对调用的检查

ContentProvider.java 是运行在服务端应用程序的应用框架类，通过 part1 的权限检查后，客户端会得到建立通信的 Binder 句柄，而 ContentProvider.java 这里会执行最终的权限检查。

和 part1 相比，权限检查很相似。但一个**大差别**是 AMS 是更宽松的，当组件级权限是空的情况直接允许（这条件返回 CP Binder 给客户端没毛病），而 ContentProvider.java 则严格，当组件级权限为空，path 权限也要为空这才 OK。

#### 不同接口不同权限检查要求

参考 [Content Provider 通信机制](#content-provider-通信模型)，这部分权限检查是由 ContentProvider 内部持有的 Binder 接口实现类（Transport）中执行的，只有 Transport 执行权限校验后才会调用服务端实现的 ContentProvider 业务抽象方法（query、delete等等）。而  readPermission/writePermission/pathPermission 权限执行在不同的 ContentProvider 业务方法中不同。举例来说：

- [Transport.query()](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-11.0.0_r25/core/java/android/content/ContentProvider.java#239)

```java
 if (enforceReadPermission(callingPkg, uri, null) != AppOpsManager.MODE_ALLOWED) {
```

说明 query() 会校验读权限。

- [Transport.insert](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-11.0.0_r25/core/java/android/content/ContentProvider.java#322)

```java
if (enforceWritePermission(callingPkg, uri, null) != AppOpsManager.MODE_ALLOWED) {
```
而 insert() 会校验写权限。

- [getType](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-11.0.0_r25/core/java/android/content/ContentProvider.java#290)

getType 不会校验权限。

拓展：实际上 getType() 不仅不会执行权限校验，而且服务端组件 exported=false 时仍能被外部调用，这是因为 AMS 特别地开了小门，单独对 getProviderMimeType 进行了实现： [getProviderMimeType](https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-11.0.0_r25/services/core/java/com/android/server/am/ActivityManagerService.java#8007)

完整方法对应的权限要求参见后表。

#### 权限检查接口实现

enforceReadPermission/enforceWritePermission() 的主要实现在 enforceReadPermissionInner()/enforceWritePermissionInner()

```java
ContentProvider.java

    protected int enforceWritePermissionInner(Uri uri, String callingPkg, IBinder callerToken)
            throws SecurityException {
        final Context context = getContext();
        final int pid = Binder.getCallingPid();
        final int uid = Binder.getCallingUid();
        String missingPerm = null;
        int strongestMode = MODE_ALLOWED;
        if (UserHandle.isSameApp(uid, mMyUid)) {
            return MODE_ALLOWED;
        }
        // 如果 exported=true，好，来看看权限有满足的情况不。
        if (mExported && checkUser(pid, uid, context)) {
            final String componentPerm = getWritePermission();
            // 权限显式满足（非空且客户端匹配权限），ALLOW！,否则再看看 path 权限，有没有机会。
            if (componentPerm != null) {
                final int mode = checkPermissionAndAppOp(componentPerm, callingPkg, callerToken);
                if (mode == MODE_ALLOWED) {
                    return MODE_ALLOWED;
                } else {
                    missingPerm = componentPerm;
                    strongestMode = Math.max(strongestMode, mode);
                }
            }
            // path 权限显式满足，ALLOW！；
            // path 权限为空，且组件权限也为空，说明 CP 没有安全要求，ALLOW！。否则再看看动态 URI 权限有没有权限。
            // track if unprotected write is allowed; any denied
            // <path-permission> below removes this ability
            boolean allowDefaultWrite = (componentPerm == null);
            final PathPermission[] pps = getPathPermissions();
            if (pps != null) {
                final String path = uri.getPath();
                for (PathPermission pp : pps) {
                    final String pathPerm = pp.getWritePermission();
                    if (pathPerm != null && pp.match(path)) {
                        final int mode = checkPermissionAndAppOp(pathPerm, callingPkg, callerToken);
                        if (mode == MODE_ALLOWED) {
                            return MODE_ALLOWED;
                        } else {
                            // any denied <path-permission> means we lose
                            // default <provider> access.
                            allowDefaultWrite = false;
                            missingPerm = pathPerm;
                            strongestMode = Math.max(strongestMode, mode);
                        }
                    }
                }
            }
            // if we passed <path-permission> checks above, and no default
            // <provider> permission, then allow access.
            if (allowDefaultWrite) return MODE_ALLOWED;
        }
        // exported 为 false？没关系，最后一次机会，动态 URI 授权。
        // last chance, check against any uri grants
        if (context.checkUriPermission(uri, pid, uid, Intent.FLAG_GRANT_WRITE_URI_PERMISSION,
                callerToken) == PERMISSION_GRANTED) {
            return MODE_ALLOWED;
        }
        // If the worst denial we found above was ignored, then pass that
        // ignored through; otherwise we assume it should be a real error below.
        if (strongestMode == MODE_IGNORED) {
            return MODE_IGNORED;
        }
        final String failReason = mExported
                ? " requires " + missingPerm + ", or grantUriPermission()"
                : " requires the provider be exported, or grantUriPermission()";
        throw new SecurityException("Permission Denial: writing "
                + ContentProvider.this.getClass().getName() + " uri " + uri + " from pid=" + pid
                + ", uid=" + uid + failReason);
    }
```

对于每个客户端的请求，StubContentProvider 都会做权限和 AppOps 检查，检查失败时要拒绝访问。检查遵照如下逻辑：

0. 默认禁止调用者访问（权限检查是失败），但调用者有以下若干机会获得允许。按顺序排查，当某当任意条件检查显示客户端具有权限时返回检查成功。

1. 检查请求是否来自同一个 UID，如果相同**检查成功**。

2. 检查组件是否对外暴露（ exported 属性），如果对外暴露，执行下一步的权限以及关联的 AppOps 检查，如果不对外暴露进入 5 的检查：

3. 根据接口类型检查调用者是否满足目标组件声明的 readPermission 或 writePermission  权限，同时检查权限关联的 AppOps，只有权限显式满足（权限非空且调用者满足要求）才**检查成功**。否则执行下一步的 Path 权限检查。

    注：这和文档中 [Path优先于组件](https://developer.android.com/guide/topics/providers/content-provider-creating.html#implementing-permissions)（「Also, path-level permission takes precedence over provider-level permissions.」）  描述有所差异（解释：这句话的意思应该是如果 provider 组件 level 的权限是不满足时，path 权限此时才优先生效），**如果 provider 组件级的权限已经满足，就不再检查 path 权限**

4. 查询调用者要访问的 URI 对应匹配的目标组件 Path，检查其声明的 readPermission 或 writePermission 权限以及权限关联的 AppOps：如果匹配的 Path 声明了指定的权限，调用者显式满足权限声明要求，返回**检查成功**如果 URI 没有匹配任何 Path 或匹配但 Path 声明的权限为空，并且第 3 步组件级权限也为空，这说明目标 CP 是无任何权限限制的公开组件，此时也返回**检查成功**。否则进入下一步检查，看看还有没有机会。

5. 到现在还是不通过？没关系，还有最后一次机会，检查调用者是否被动态授予了目标 URI 权限

    注：当 URI 对应的 ContentProvider exported = true 时 Context.checkUriPermission 方法始终返回 true。

从代码看前两步时调用 checkPermissionAndAppOp 来实现的，展开分析这个方法：

```java
    /**
     * Verify that calling app holds both the given permission and any app-op
     * associated with that permission.
     */
    private int checkPermissionAndAppOp(String permission, String callingPkg,
            IBinder callerToken) {
        if (getContext().checkPermission(permission, Binder.getCallingPid(), Binder.getCallingUid(),
                callerToken) != PERMISSION_GRANTED) {
            return MODE_ERRORED;
        }
        return mTransport.noteProxyOp(callingPkg, AppOpsManager.permissionToOpCode(permission));
    }
```
观察 checkPermissionAndAppOp 方法，可以发现分别调用 Context.checkPermission() 和 AppOpsManager.noteProxyOp()，并要求二者同时满足要求。（由此应证，Context.checkPermission 只检查 permission 而不检查 AppOps）

```java
    /**
     * Verify that calling app holds both the given permission and any app-op
     * associated with that permission.
     */
    private int checkPermissionAndAppOp(String permission, String callingPkg,
            IBinder callerToken) {
        if (getContext().checkPermission(permission, Binder.getCallingPid(), Binder.getCallingUid(),
                callerToken) != PERMISSION_GRANTED) {
            return MODE_ERRORED;
        }
        return mTransport.noteProxyOp(callingPkg, AppOpsManager.permissionToOpCode(permission));
    }
```
Context.checkPermission() 这里不细说，就是进入 AMS/PMS 来执行传统的「权限」检查。重点说下 noteProxyOp()。

noteProxyOp 的最终实现在 [AppOpsService.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/services/core/java/com/android/server/appop/AppOpsService.java)，AppOpsService 是运行在 system_server 中的一个系统服务，noteProxy() 会检查 permission 关联的 op 是否被允许。这里 ContentProvider 调用的是 noteProxyOp() 而不是 noteOp()，因此要求 proxy 方（Content Provider 提供方）和 proxied 方（Content Provider 调用方）方同时满足 op 要求。详细内容参考 AppOpsService。

注：exported 的默认值，API 17 即以后是 true，在此之前是没有这个属性的，所以默认行为等价于是 true。[参见](https://developer.android.com/guide/topics/manifest/provider-element)

## authorities

在清单文件 Content Provider 元素中类名（name）仅标识实现类，而 authorities 才是唯一标识（因此，不同应用定义相同 authorities 会导致冲突，而无法安装）。实现类通过 [addURI](https://developer.android.com/guide/topics/providers/content-provider-creating#content-uri-patterns) 方法向指定 authorities 添加 path （组合形成 URI）和对应处理代码、或 FileProvider 的 [getUriForFile](https://developer.android.com/reference/androidx/core/content/FileProvider#getUriForFile(android.content.Context,%20java.lang.String,%20java.io.File)) 来向指定 authorities 添加 URI （文件资源）
